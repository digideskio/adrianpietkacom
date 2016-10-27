# Doctrine ORM & Repositories

Abstrakcyjne repozytorium udostępnine poprzez Doctrine ORM jest bardzo atrakcyjne pod wzgędem dostarczonej funkcjonalności. Wystarczy wywołać metodę ```getRepository``` na obiekcie Entity Managera aby otrzymać obiekt repozytorium ```Doctrine\ORM\EntityRepository``` implementujacy dwa interfejsy:

```
Doctrine\Common\Persistence\ObjectRepository
```

oraz

```
Doctrine\Common\Collections\Selectable
```

Przykład utworzenia repozytorium na podstawie UserEntity:

```php
$userRepository = $entityManager->getRepository(UserEntity::class);
```

Autorzy Doctrine ORM zaimplementowali nieco więcej metod niż te wymuszone w.w interfejsami. Wśród nich znajduje się jednak prawdziwa perełka - metoda magiczna ```__call()```. Dzięki niej otrzymujemy możliwość używania niezdefiniowanego nigdzie interfejsu - chyba, że mamy na myśli schemat tabeli w bazie danych (ilość udostępnionych *metod* = ilość kolumn w tabeli bazodanowej * 2).
Konkretniej, implementacja umożliwia nam wywoływanie nieistniejących metody w oparciu o schemat: ```findOneBy[ColumnName]``` oraz ```findBy[ColumnName]```. Rozwiązanie wydaje się bardzo atrakcyjne dla programisty, "od tak" otrzymujemy możliwość wyszukiwania encji po wskazanym kryterium.

Okazuje się jednak, że z początkowych publicznych metod, których doliczyłem się w klasie ```EntityRepository``` dokładnie **15**, nasz końcowy obiekt repozytorium utworzony dla konkretnej encji (zawierającej 10 właściwości) zawiera ich teoretycznie **35** ( ```15 + (10*2)```). Teoretycznie bo 20 z nich jest wynikiem implementacji metody ```__call()```. Podsumowując: 20 nigdzie nie zdefiniowanych metod - obsługiwanych w magiczny sposób. 

Często spotykam się z mniej więcej takimi rozwiązaniami:

```php
namespace Services;

use Exception\DuplicatedUserEmailException;
use Entity\UserEntity;
use Doctrine\ORM\EntityRepository;

class CreateNewUserService
{
    private $userRepository;
    
    public function __construct(EntityRepository $userRepository)
    {
        $this->userRepository = $userRepository;
    }
    
    public function create(string $email)
    {
        // ...
        
        if ($this->userRepository->findByAddessEmail($email)) {
            throw DuplicatedUserEmailException();
        }
        
        $this->userRepository->add(new UserEntity($email));
    }
}
```

```php
$entityManager = \Doctrine\ORM\EntityManager::create();
$userRepository = $entityManager->getRepository(\Entity\UserEntity::class);
$createNewUserService = new \Services\CreateNewUserService($userRepository);

$createNewUserService->create('unikalnyadres@pocztaemail.com');
```

Wykorzystana w przykładzie metoda ```findByAddessEmail()``` jest wynikiem w.w magii Doctrina. Dodatkowo jesteśmy zależnieni od zewnętrznego interfejsu ```Doctrine\ORM\EntityRepository```, z niewiadomą ilością wywołań (w naszym kodzie) niezdefiniowanych metod interfejsu. Próba podmiany implementacji repozytorium, to po prostu walka z kodem po omacku.

Podsumowując - bezpośrednie wykorzystywanie takiego obiektu repozytorium, bez przykrycia go warstwą własnej abstrakcji, jest dla mnie nie do przyjęcia ze względu na:

- brak sprecyzowanego komunikatu - z jakiego konkretnego repozytorium chcemy skorzystać, możemy oczekiwać repozytorium encji użytkownika, a w rzeczywystości zostanie nam przekazane repozytorium komentarzy, oczekujemy tutaj bardzo generycznego obiektu klasy ```EntityRepository```,
- chęć korzystania z kontraktów (interfejsów) jest tutaj dość kłopotliwa, ponieważ konkretna implementacja dostarcza nam bardziej wzbogacone *API* (więcej metod) niż ta zdefiniowana w interfejsach,
- korzystanie z magicznych rozwiązań utrudnia w tym wypadku podmianę implementacji, o ile w ściśle zdefiniowany interfejsie wiemy jakie metody musimy zaimplementować o tyle tutaj, bez testów jednostkowych lub/i przeglądu kodu nie jesteśmy w stanie tego wykonać.

## Warstwa abstrakcji

Mając na uwadze powyższe wady bezpośredniego wykorzystania gotowego rozwiązania, pokusiłem się o implementację prostej warstwy abstrakcji. Wykorzystuje ona Doctrinowe ```getRepository()```, lecz opakowuje faktycznie wykorzystywane w aplikacji metody, ukrywając tym samym szczegóły implementacyjne repozytorium.

Zrzut struktury plików wygląda następująco:

```
Repository
|-- Doctrine
|   |-- UserRepository.php
|   |-- CommentRepository.php
|-- UserRepositoryInterface.php
|-- CommentRepositoryInterface.php
```

Folder ```Repository/Doctrine``` zawiera konkretną już implementację interfejsów zdefiniownaych w przestrzeni ```Repository```.

Rzućmy okiem na definicję interfejsu ```UserRepositoryInterface```:

```php
namespace Repository;

class UserRepositoryInterface
{
    public function getByAddessEmail(string $email) : UserEntity;
    public function add(UserEntity $userEntity) : UserEntity;
}
```

W dalszej części naszej aplikacji możemy jawnie wskazać krótego repozytorium oczekujemy - w tym wypadku należy pamiętać o "design by contract", więc oczekujemy *interfejsu repozytorium*, a nie jego implementacji.

Przykładowa implementacja:

```php
namespace Repository\Doctrine;

use Entity\UserEntity;
use Repository\UserRepositoryInterface;

class UserRepository implement UserRepositoryInterface
{
    private $entityManager;
    
    public function __construct($entityManager)
    {
        $this->entityManager = $entityManager;
    }
    
    public function getByAddessEmail(string $email) : UserEntity
    {
        return $this->entityManager->getRepository(UserEntity::class)
            ->findByAddressEmail($email);
    }
    
    public function add(UserEntity $user) : UserEntity
    {
        $this->entityManager->persist($user);
        $this->entityManager->flush();

        return $user;
    }
}
```

Następnie ```CreateNewUserService``` ulegnie drobnym modyfikacją:


```php
namespace Services;

use Repository\UserRepositoryInterface;

class CreateNewUserService
{
    private $userRepository;
    
    public function __construct(UserRepositoryInterface $userRepository)
    {
        $this->userRepository = $userRepository;
    }
    
    public function create(string $email)
    {
        // ...
        
        if ($this->userRepository->getByAddessEmail($email)) {
            throw DuplicatedUserEmailException();
        }
        
        $this->userRepository->add(new UserEntity($email));
    }
}
```

Wywołanie też wymaga nieco odmiennej definicji:

```php
$entityManager = \Doctrine\ORM\EntityManager::create();
$userRepository = new \Repository\Doctrine\UserRepository($entityManager);
$createNewUserService = new \Services\CreateNewUserService($userRepository);

$createNewUserService->create('unikalnyadres@pocztaemail.com');
```

Zmiany widoczne w kodzie są kosmetyczne, jednak takie podejście umożliwia nam łatwą podmianę repozytorium. Jawnie zdefiniowany interfejs informuje nas o metodach które faktycznie wykorzystywane są w aplikacji. 

Dziś wykorzystujemy Doctrine, jednak jutro może się okazać, że część danych migrujemy w zupełnie inny byt i potrzebujemy dane przesyłać do zewnętrznego API. Przy wyabstrahowanym rozwiązaniu potrzebujemy jedynie zdefiniować nowy namespace np. ```Repository/WebService```, zaimplementować interfejs w oparciu o inne źródło danych (np. z wykorzystaniem *Guzzle*), podmienić moment wstrzyknięcia implementacji do klasy ```CreateNewUserService``` z implementacji Doctrinowej na nową.

## Podsumowanie

Aby było jasne. Nie twierdzę, że repozytorium dostarczone przez twórców Doctrine ORM jest złe. Źle jest ono zazwyczaj wykorzystane w realnym projekcie. Pokusa bezpśredniego wykorzystania Doctrinowego repozytorium jest o tyle większa, że programista otrzymuje potężny *interfejs* do działania na kolekcji danego typu encji m.in za pomocą metod ```findBy*```, ```findOneBy*```, ```matching```. Szybkość zaimplementowania kolejnej funkcjonalności często przegrywa z dobrymi praktykami, a w tym wypadku po prostu z abstrakcją którą na porządku dziennym powinniśmy wykorzystywać w programowaniu obiektowym. Odbija się to czkawką gdy słyszymy od klienta o migrowaniu części danych do innego zasobu. Jednak ten problem poruszę innym razem.