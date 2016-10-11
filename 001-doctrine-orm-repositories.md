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

Autorzy Doctrine ORM zaimplementowali metody wymuszone w.w interfejsami oraz dodali kilka kolejnych...

Jednak prawdziwą perełką jest dla mnie metoda magiczna ```__call()```. Dzięki niej otrzymujemy możliwość używania niezdefiniowanego nigdzie interfejsu - chyba, że mamy na myśli schemat tabeli w bazie danych (ilość udostępnionych *metod* = ilość kolumn w tabeli bazodanowej* 2).
Konkretniej, implementacja umożliwia nam wywoływanie nieistniejących metody w oparciu o schemat: ```findOneBy[ColumnName]``` oraz ```findBy[ColumnName]```.

Nagle okazuje się, że z początkowych publicznych metod, których doliczyłem się w klasie ```EntityRepository``` dokładnie **15**, nasz końcowy obiekt repozytorium utworzony dla konkretnej encji zawierającej 10 właściwości zawiera ich **35** ( ```15 + (10*2)```), z czego 20 z nich jest wynikiem implementacji metody ```__call()```.

Podsumowując - takie posługiwanie się abstrakcyjnymi repozytoriami (w podejściu, że traktujemy wytworzone obiekty jako prawdziwe repozytoria), jest dla mnie nie do przyjęcia ze względu na:

- brak sprecyzowanego komunikatu - z jakiego konkretnego repozytorium chcemy skorzystać, możemy oczekiwać repozytorium encji użytkownika, a w rzeczywystości zostanie nam przekazane repozytorium komentarzy,
- chęć korzystania z kontraktów (interfejsów) jest tutaj dość kłopotliwa, ponieważ konkretna implementacja dostarcza nam bardziej wzbogacone API (więcej metod) niż ta zdefiniowana w interfejsach, musimy się opierać na *pseudo*kontrakcie opartym o konkretną już implementację,
- korzystanie z magicznych rozwiązań utrudnia w tym wypadku podmianę implementacji, o ile w ściśle zdefiniowany interfejsie wiemy jakie metody musimy zaimplementować o tyle tutaj, bez testów jednostkowych lub/i przeglądu kodu nie jesteśmy w stanie tego wykonać.

## Warstwa abstrakcji

Mając na uwadze powyższe wady gotowego rozwiązania, pokusiłem się o implementację własnej warstwy repozytorium.

Zrzut struktury plików wygląda następująco:

```
Repository
|-- Doctrine
|   |-- UserRepository.php
|   |-- CommentRepository.php
|-- UserRepositoryInterface.php
|-- CommentRepositoryInterface.php
```

Folder ```Repository/Doctrine``` zawiera konkretną implementację interfejsów zdefiniownaych w przestrzeni ```Repository```.

Rzućmy okiem na definicję interfejsu ```UserRepositoryInterface```:

```php
namespace Repository;

class UserRepositoryInterface
{
    public function getById(int $userId) : UserEntity;
    public function getByEmail(string $userEmail) : UserEntity;
    public function getAll(int $limit = 10) : array;
}
```

Jawnie zdefiniowane *API* informuje nas o metodach które faktycznie wykorzystywane są w aplikacji. Umożliwia to między innymi szybszą implementację pełnego interfejsu.

Jeżeli potrzebujemy zaimplementować inne źródło danych (np. dowolny typ WebService) wystarczy zdefiniować nowy namespace ```Repository/WebService``` i umieścić konkretną implementację dla danego interfejsu.

Przy dobrze zaprojektowanej architekturze aplikacji nie będziemy potrzebowali wprowadzania modyfikacji w kodzie aby dokonać podmiany implementacji. Jeżeli działamy w oparciu o framework Symfony 2.x/3.x wystarczy aby nową implementację wstrzyknąć (Dependency Injection) w odpowiednie cześci aplikacji.

## Podsumowanie

Nie twierdzę, że repozytorium dostarczone przez twórców Doctrine ORM jest złe, lecz złe jest zazwyczaj ich wykorzystanie w realnym projekcie. Pokusa ich bezpśredniego wykorzystania jest o tyle większa, że programista otrzymuje potężny *interfejs* do działania na kolekcji danego typu encji m.in za pomocą metod ```findBy```, ```findOneBy```, ```matching```. Szybkość zaimplementowania kolejnej funkcjonalności często przegrywa z dobrymi praktykami, a w tym wypadku po prostu z abstrakcją którą na porządku dziennym powinniśmy wykorzystywać w programowaniu obiektowym. Odbija się to czkawką gdy słyszymy od klienta o migrowaniu części danych do innego zasobu. Jednak ten problem poruszę innym razem. 