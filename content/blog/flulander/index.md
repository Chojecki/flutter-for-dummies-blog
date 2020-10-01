---
title: Flulander cz. 1 / 2
date: "2020-09-29"
description: "Trendi as fak"
---

W tym artykule/tutorialu czy jak to zwać, chciałem pokazać wam jak zbudować turbo prostą apkę, polegającą na zalogowaniu lub zarejestrowaniu się do naszej apki za pomocą FirebaseAuth. Jeżeli byłby to tutorial na YT to jego tytuł brzmiałby “Flutter Firebase Authentication” - czyli już wiadomka ocb. Brzmi prosto, ale moim zdaniem będzie to materiał dla osób średniozaawansowanych ponieważ pokażę wam moje podejście do tego tematu, a więc:

- Domain Driven Design / Clear Architecture
- Riverpod <3
- Freezed
- StateNotifer
- Dartz
- Hooki
- Testy riverpod
- Inne takie takie
- Na koniec CI za pomocą Github Actions

> Nie jest to tutorial nt Freezed, Riverpod itp. a raczej tutorial jak to ze sobą połączyć w prawdziwej apce. Wszelkie hello wordy tych paczek są dobrze opisane w dokumentacjach.
>
> Zaprezentowane podejście jest
> a) Inspirowane tym co znajdziecie w kursie Resocodera DDD.
> b) Nie będę tłumaczył takich rzeczy jak skonfigurowanie Firebase dla Fluttera czy jak się instaluje paczkę pub (tak jak wspomniałem, myślę, że nie jest to materiał dla początkujących plus takie rzeczy najlepiej opisane są w dokumentacji.
> c) Używamy Riverpod który wg jego twórcy już mocno się nie zmieni, ale to dalej stabilna beta plus sam nie jestem kotem w tej technologii, więc jest to moje podejście i jestem otwarty na uwagi.

### Dobra to zaczynamy:

#### DDD:

> Wikipedia powie wam, że Domain-driven design – podejście do tworzenia oprogramowania kładące nacisk na takie definiowanie obiektów i komponentów systemu oraz ich zachowań, aby wiernie odzwierciedlały rzeczywistość. Dopiero po utworzeniu takiego modelu należy rozważyć >zagadnienia związane z techniczną realizacją.

No i spoko, ale mi na początku dużo to nie mówiło (w sumie dalej w DDD nie czuje się turbo mocny). Jeżeli obejrzycie materiał na ten temat z Flutter Europe, to możliwe, że tak jak ja, też poczujecie się przytłoczeniu natłokiem nowych informacji. Dlatego tuaj mamy DDD for dummies. Myślę, że każdy poważny Inżynier a nie klepacz kodu (ja jestem raczej klepaczem) będzie miał tu coś do dodania i pewnie słusznie, ale może na początek wystarczy moje łopatologiczne podejście.

W skrócie projekt (to co jest w /lib) podzielimy sobie na 4 główne foldery: domain, infrastructure, application, presentation. Poza tymi folderami w /lib będzie tylko main.dart. I w sumie tyle. Możecie sobie nazywać wasze foldery w ten sposób i już czuć się jak Wujek Bob. Kolejność nie jest przypadkowa i odzwierciedla flow tworzenia ficzerów w apce (pomijam testowanie jeżeli ktoś pisze z TDD, my potestujemy na końcu aby było klarowniej). Cały ten proces fajnie odzwierciedla jakieś takie ala zwinne flow i przechodzimy od największej abstrakcji do implementacji na widokach.

#### Domain

Czyli mamy jakąś domenę biznesową, np. Analityk/PM/Ktoś mówi “Yo, apka ma mieć system autoryzacji usera”. Nie ma tu mowy o tym czego użyjemy. Jakiego frameworka? Jaki backend? Etc. Jest prosta domena “Ma być auth”. Więc my w folderze domain tworzymy dla foldery “core” i “auth” (core jest zawsze a auth to nazwa naszej domeny autoryzacji - w następnych folderach będziemy to nazewnictwo powtarzać).

I tutaj zaczyna się zabawa bo wchodzimy to domain/auth/core i tworzymy sobie plik `failures.dart` … ale jak to? Ktoś zapyta. No tak to, że zaczynamy od zdefiniowania sobie jakie możemy dostać błędy po stronie aplikacji (nie serwera) podczas logowania. Np. email nie jest typu email i wasze dupa123 nie pasuje do wzoru dupa@123.pl wiec User otrzyma błąd, że format email jest zły. Użyjemy tutaj paczki freezed, która jest generatorem kodu (jeżeli ktoś nie zna build_runnera do generowania kodu we Flutterze, bo jest dużo materiałów na np YouTubie i jak to mówią modni tutorialowcy - it’s out of scope of this tutorial). A więc nasze failures to:

```dart
part 'failures.freezed.dart';

@freezed
abstract class ValueFailure<T> with _$ValueFailure<T> {
  const factory ValueFailure.exceedingLength({
    @required T failedValue,
    @required int max,
  }) = ExceedingLength<T>;
  const factory ValueFailure.empty({
    @required T failedValue,
  }) = Empty<T>;
  const factory ValueFailure.invalidEmail({
    @required T failedValue,
  }) = InvalidEmail<T>;
  const factory ValueFailure.shortPassword({
    @required T failedValue,
  }) = ShortPassword<T>;
}
```

Ich nazwy jak “invalid email” mówią same za siebie co oznaczają.

Aby móc sobie walidować naszego emaila i password, trzeba stworzyć jakieś walidatory, a więc dalej w domain/core tworzymy value validators.dart. Używamy tutaj paczki dartz, która w skrócie przynosi świat functional programming do fluttera i daje nam takie dobra jak typ `Either<Left, Right>` który może zwracać Left lub Right w zależności od “ustawień”. Każdy z naszych validatorów zwróci albo ValueFailure albo String (bo validujemy email i password). Czyli walidator przyjmuje String input, mieli to, i zaraca albo right czyli “Elegancko, ten String jest gitara” i zwraca ten sam string albo left czyli “_halo wpisałeś dupa123 a nie dupa@123.pl, oto twój błąd w postaci ValueFailure_”.

```dart
import 'package:dartz/dartz.dart';

Either<ValueFailure<String>, String> validateMaxStringLenght(String input, int maxLength) {
  if (input.length <= maxLength) {
    return right(input);
  } else {
    return left(ValueFailure.exceedingLength(failedValue: input, max: maxLength));
  }
}

Either<ValueFailure<String>, String> validateStringNotEmpty(String input) {
  if (input.isNotEmpty) {
    return right(input);
  } else {
    return left(ValueFailure.empty(failedValue: input));
  }
}

Either<ValueFailure<String>, String> validateEmailAddress(String input) {
  const emailRegex = r"""^[a-zA-Z0-9.a-zA-Z0-9.!#$%&'*+-/=?^_`{|}~]+@[a-zA-Z0-9]+\.[a-zA-Z]+""";
  if (RegExp(emailRegex).hasMatch(input)) {
    return right(input);
  } else {
    return left(ValueFailure.invalidEmail(failedValue: input));
  }
}

Either<ValueFailure<String>, String> validatePassword(String input) {
  if (input.length >= 6) {
    return right(input);
  } else {
    return left(ValueFailure.shortPassword(failedValue: input));
  }
}
```

Dalej tworzymy errors.dart i jest to po prostu typ Errora dla wartości (można śmiało kopiować i potem przy używaniu się rozjaśni ocb)

```dart
import 'package:bangerify/domain/core/failures.dart';

class NotAuthenticatedError extends Error {}

class UnexpectedValueError extends Error {
  final ValueFailure valueFailure;

  UnexpectedValueError(this.valueFailure);

  @override
  String toString() {
    const explanation = 'Encountered a ValueFailure at an unrecoverable point. Terminating.';
    return Error.safeToString('$explanation Failure was: $valueFailure');
  }
}

```

Dalej wsadzimy tutaj koncept Value Object o którym możecie sobie pooglądać [tutaj](https://www.youtube.com/playlist?list=PLB6lc7nQ1n4iS5p-IezFFgqP6YvAJy84U). Ponownie for dummies jak ja: Nasze obiekty będą trzymały jakąś wartość - czyli obiekt “email” nie będzie typu String ale typu `EmailAddress`, z kolei `EmailAddress` będzie właśnie rozszerzał ten `ValueObject` z core (dlatego jest w core). Po co? A no po to, że taki obiekt może się np auto walidować (polecam serio tutki od Resocodera w tym temacie). Czyli nasz `value_object.dart` wygląda tak:

```dart
import 'package:dartz/dartz.dart';
import 'package:flutter/foundation.dart';
import 'package:uuid/uuid.dart';

@immutable
abstract class ValueObject<T> {
  const ValueObject();
  Either<ValueFailure<T>, T> get value;

  /// Throws [UnexpectedValueError] containing the [ValueFailure]
  T getOrCrash() {
    // id = identity - same as writing (right) => right
    return value.fold((f) => throw UnexpectedValueError(f), id);
  }

  Either<ValueFailure<dynamic>, Unit> get failureOrUnit {
    return value.fold(
      (l) => left(l),
      (r) => right(unit),
    );
  }

  bool isValid() => value.isRight();

  @override
  bool operator ==(Object o) {
    if (identical(this, o)) return true;

    return o is ValueObject<T> && o.value == value;
  }

  @override
  int get hashCode => value.hashCode;

  @override
  String toString() => 'Value($value)';
}
```

Ponownie dartz w grze. Wartość naszego obiektu będzie właśnie typu `Either<AuthFailure, Right>`. Czyli ponowne nasz email, który w 99% tutoriali będzie `Stringiem`, u nas będzie EmailAddressem o wartości `Either<ValueFailure<String>, String>>` czyli zwróci albo błąd ze `Stringiem` albo `String`. Zaraz dojdziemy do tworzenia konkretnych ValueObjectów jak `EmailAddress` czy `Password`.

Na koniec nasz pierwszy obiekty rozszerzający `ValueObject` - jedyny w core bo bardzo uniwersalny, czyli `UniqueId`.

```dart
class UniqueId extends ValueObject<String> {
  @override
  final Either<ValueFailure<String>, String> value;

  factory UniqueId() {
    return UniqueId._(right(Uuid().v1()));
  }

  factory UniqueId.fromUniqueString(String uniqueId) {
    assert(uniqueId != null);
    return UniqueId._(right(uniqueId));
  }

  const UniqueId._(this.value);
}
```

Jak widzisz jako, że rozszerza `ValueObject<String>` to jego value będzie `Either<ValueFailure<String>, String>` czyli albo błąd albo `String`.

Ma też dwie fabryki jedna do tworzenia go ze z góry znanym ID (np ten z firebase) druga jeżeli nie podamy ID to sam sobie je stworzy za pomocą paczki `Uuid`.

Uff core skończony. Jak widzicie nawet dobrze nie zaczęliśmy a już wszystko jest pokręcone jak lato z radiem.

Ale ok jesteśmy dalej w Domain ale tworzymy folder (nie plik) “auth”. Yep zaczynamy projektować domenę autoryzacji!

Normalnie tworzymy jakiś model typu `User` prawda? No to pyk, nie ma chuja, tutaj tworzymy `Entity`! Jep `Entity` to taka jednostka, którą modele dopiero mogą implementować. To coś turbo abstrakcyjnego czyli potwierdza się to co pisałem wcześniej - jesteś w domain smarkaczu, tutaj tylko filozofujemy! A więc filozofujemy, że w sumie w autoryzacji to autoryzujemy jakiegoś `Usera` i tworzymy `user.dart` oczywiście będzie to klasa zrobiona za pomocą `Freezed` bo kochamy `Freezed`.

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart';

@freezed
abstract class User with _$User {
  const factory User({
    @required UniqueId id,
  }) = _User;
}
```

Aby nie gmatwać nie wiadomo jak, nasza jednostka usera ma tylko ID typu uwaga… `UniqueId` - yep w końcu użyliśmy tego syfu z `domain/code`. Widzicie jakie to posrane - normalnie jest model `User` który ma id typu `String` a tu jakiś typ wam mówi, że macie `Entity User`, który ma `ID` typu `UniqueId` które ma wartość `String`. Jeżeli ten koncept nie wchodzi wam teraz do głowy, to warto się nad tym jeszcze zastanawiać, bo wyraźnie widać wzór.

Dalej robimy to samo co w core czyli tworzymy błędy i value object ale tym razem już stricte pod “auth”. Tak jak w core robiliśmy sobie `failures.dart` które były ogólne - taki `invalidEmail` nie musi być zastosowany tylko w autoryzacji bo później np można mieć domene wysyłania emaila itp. Tutaj już jesteśmy w domain/auth więc będą to auth_failures.dart. Też tworzone za pomocą Freezed (bo to Union - klasa freezed z kilkoma fabrykami tak łopatologicznie). Błędy te to typowe błędy z firebase przy autoryzacji jak “email jest już używany”

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'auth_failure.freezed.dart';

@freezed
abstract class AuthFailure with _$AuthFailure {
  const factory AuthFailure.serverError() = ServerError;
  const factory AuthFailure.emailAlreadyInUse() = EmailAlreadyInUser;
  const factory AuthFailure.invalidEmailAndPasswordCombination() = InvalidEmailAndPasswordCombination;
}
```

Mamy błędy to dalej jak w core ValueObject. W Auth ValueObjecty będą dwa czyli wspominany już `EmailAddress` i `Password`. Tworzymy je dokładnie tak jak poprzednio `UniqueId`.

```dart

class EmailAddress extends ValueObject<String> {
  @override
  final Either<ValueFailure<String>, String> value;

  factory EmailAddress(String input) {
    assert(input != null);
    return EmailAddress._(validateEmailAddress(input));
  }

  const EmailAddress._(this.value);
}

class Password extends ValueObject<String> {
  @override
  final Either<ValueFailure<String>, String> value;

  factory Password(String input) {
    assert(input != null);
    return Password._(
      validatePassword(input),
    );
  }

  const Password._(this.value);
}
```

Ba nawet prościej, bo tutaj jest tylko wartość value (typu błąd ze Stringiem albo `String`) i jedna fabryka, która po prostu waliduje input.

#### STOP (ten mikrofon to glock)

Zastanów się teraz przez chwilę nad samą koncepcją ValueObjectu i czy ma to dla Ciebie sens (nie musi mieć, bo to tylko koncept a nie elementarna wiedza). Jednak przemyśl czy to wszystkie kropki jakoś się łączą:
mamy koncept value object jakiegoś typu czyli obiektu, który ma jakąś wartość błędu lub właśnie tego typu (Either - to albo to)
Mamy walidatory np. `emailValidator` które też zwracają wartość `Either<błąd, string>`
Mamy teraz value object - obiekt `EmailAddress`, który ma fabrykę gdzie przekazujemy String i ta fabryka za pomocą walidatora zwraca błąd albo string, czyli taki sam typ jaki ma value `EmailAddress`.
Czyli tworząc sobie jakiś obiekt `final adres = EmailAddress(‘dupa’)`, `adres.value` będzie błędem i to konkretnym błędem jaki sobie wcześniej stworzyliśmy - czyli zły format adresu, a tworząc `final adres = EmailAddress(‘dupa@123.pl)` , `adres.value` będzie Stringiem dupa@123.pl - Czy to zaczyna mieć sens?

Karawana jedzie dalej.

Ostatnie co tworzymy w `domain/auth` to dumnie brzmiące `IAuthFacade` - fasada to taki wzorzec projektowy i tak jak mówiłem, jestem klepaczem kodu a nie inżynierem, więc ciężko mi określić jaka jest dokładnie różnica pomiędzy Fasadą, Serwisem, Repozytorium. Ogólnie koncept jest Ci pewnie znany jeżeli nie jesteś Juniorką/Juniorem i chodzi o to, że mamy “coś” co wstrzykujemy “gdzieś” i z pomocą tego “czegoś” w tym “czymś” robimy najczęściej zapytania do API :) Ale tutaj mamy literkę “I” przed nazwą i pamiętajmy - Domain - tutaj tylko filozofujemy, więc to będzie nasz abstrakcyjny interface. Czyli taki taki manager w firmie Pol-TransEx, który nic nie robi, a jedynie mówi co ma być zrobione i jaki ma być wynik tej pracy.

```dart
abstract class IAuthFacade {
  Future<Option<User>> getSignedInUser();
  Future<Either<AuthFailure, Unit>> registerWithEmailAndPassword(
      {@required EmailAddress emailAddress, @required Password password});

  Future<Either<AuthFailure, Unit>> signInWithEmailAndPassword({
    @required EmailAddress emailAddress,
    @required Password password,
  });

  Future<void> signOut();
}

```

Widzicie, słowo `“abstract”` to jest ten klucz właśnie. Tylko nazwy metod i tego co zwracają. Zero implementacji. Tak sobie filozofujemy, że ta domena autoryzacji to w sumie ma już Usera ale z tym Userem to możemy go zarejestrować, zalogować, wylogować i uzyskać (np. aby wyświetlić).

Uwaga, skończyliśmy folder `“domain”` 25% roboty za nami!

#### INFRASTRUCTURE

Przechodzimy do `lib/infrastructure`. Ta warstwa apli w DDD to już nie filozofowanie ale implementacja. Bardzo łopatologicznie w infrastructure głównie (ale nie tylko) implementujemy serwisy/fasady i inne takie takie.

Najpierw jednak zrobimy szybki mapper o nazwie `firebase_user_mapper.dart`

```dart
extension FirebaseUser on firebase.User {
  User toDomain() {
    return User(id: UniqueId.fromUniqueString(uid));
  }
}
```

Po co? Określiliśmy sobie jednostkę Entity `User`. Ma on `ID` typu `UniqueId`, a Firebase zwróci nam swój typ `User`, który ma id `String`. Potrzebujemy więc mappera, który po prostu weźmie takiego Usera z Firebase i zrobi z niego naszego `Usera` zdefiniowanego w domain.

Ok, czyli dalej, tak jak w `domain/auth` zrobiliśmy `I_auth_facase.dart` tak tutaj zbudujemy tego pracownika Pol-TransExu, który robi już wszystko to co określił manager (`IAuthFacade`). Czyli `class Pracownik implements Manager`. Zanim rzucisz okiem na kod, to po co takie świrowanie? A no po to, że teraz użyjemy Firebase i zrobimy `FirebaseAuthFacade`, ale jak za rok super trendy będzie `IceBase` czy coś innego to robisz `IceBaseFacade`, które implementuje `IAuthFacde` i reszta apki działa tak samo jak wcześnej, bo to Manager `IAuthFacase` decyduje co ma być zrobione a implementacja określa tylko jak. Będzie to też pomocne przy testowaniu.

```dart
import 'package:dartz/dartz.dart';
import 'package:firebase_auth/firebase_auth.dart' hide User;
import 'package:flutter/foundation.dart';
import 'package:riverpod/all.dart';

import './firebase_user_mapper.dart';

final firebaseAuthProvider = Provider((ref) => FirebaseAuth.instance);

final firebaseAuthFacadeProvider = Provider<IAuthFacade>((ref) => FirebaseAuthFacade(ref.read));

class FirebaseAuthFacade implements IAuthFacade {
  FirebaseAuthFacade(this._read);

  final Reader _read;
  FirebaseAuth get auth => _read(firebaseAuthProvider);

  @override
  Future<Option<User>> getSignedInUser() async {
    return optionOf(auth.currentUser?.toDomain());
  }

  @override
  Future<Either<AuthFailure, Unit>> registerWithEmailAndPassword(
      {@required EmailAddress emailAddress, @required Password password}) async {
    final emailAddressStr = emailAddress.getOrCrash();
    final passwordStr = password.getOrCrash();

    try {
      await auth.createUserWithEmailAndPassword(email: emailAddressStr, password: passwordStr);
      return right(unit);
    } catch (e) {
      if (e.code == 'email-already-in-use') {
        return left(const AuthFailure.emailAlreadyInUse());
      } else {
        return left(const AuthFailure.serverError());
      }
    }
  }

  @override
  Future<Either<AuthFailure, Unit>> signInWithEmailAndPassword(
      {EmailAddress emailAddress, Password password}) async {
    final emailAddressStr = emailAddress.getOrCrash();
    final passwordStr = password.getOrCrash();
    try {
      await auth.signInWithEmailAndPassword(email: emailAddressStr, password: passwordStr);
      return right(unit);
    } on FirebaseAuthException catch (e) {
      if (e.code == 'error_wrong_password' || e.code == 'ERROR_USER_NOT_FOUND') {
        return left(const AuthFailure.invalidEmailAndPasswordCombination());
      } else {
        return left(const AuthFailure.serverError());
      }
    }
  }

  @override
  Future<void> signOut() async {
    auth.signOut();
  }
}
```

Ale zaraz… wasze czujne oko Ferdynanda po wyścigu, pewnie wychwyciło słówko `“Provider”` nad klasą `FirebaseAuthFacade`. I tu was mam żuczki… instalujemy `Riverpod`! Tak ten `riverpod` o którym czytałeś. Tak od tego Remiego. Tak możesz już odpalać Twittera i zacząć pisać o tym, że Provider to może był spooko w 2020 ale Ty mentalnie jesteś w 2057.

Ale powoli i od początku. Ogólnie do autoryzacji potrebujemy `FirebaseAuth`. Jak to w Firebase to jest singleton więc tworzenie miliona instancji nie jest potrzebne. Tutuaj mamy minimalną apkę, więc pewnie nie miało by to znaczenia, ale i tak nasz `FirebaseAuth.instance` zamiast w samej klasie fasady, dostarczymy poprzez riverpod, a jak! Wiec tworzymy sobie Provider `firebaseAuthProvider`.

Dalej, nasz `FirebaseAuthFacade` musimy jakoś “dostarczać”/”wstrzykiwać” do klas odpowiedzialnych za zarządzanie stanem, więc tworzymy kolejny Provider `firebaseAuthFacadeProvier` i jest to Provider typu `IAuthFacade` czyli tej samej abstrakcyjnej klasy co to co w nim przekazujemy. Ważne jest zdefiniowanie typu, bo później w testach wstrzykniemy sobie MockFacade które też będzie typu `IAuthFacade`. Tricky jest ten ref.read. A chodzi w nim o to, że `FirebaseAuthFacade` musi dostać nasze `FirebaseAuth.instance` które też jest w Providerze. Providery mogą się czytać wzajemnie poprzez właśnie `Reader ref.read`. Więc `FirebaseAuthFacade` zamiast przyjąć w konstruktorze `FirebaseAuth`, przyjmuje `Reader _read` i za jego pomocą “dobierze się” to `firebaseAuthProvier` - ma to sens? Oczywiście nie macie jak mi odpowiedzieć, więc jadę dalej.

Te metody do logowanie, rejestracji, wylogowania czy uzyskania usera, również korzystają z dartz i `Either`. Czyli zwracają `Future<Either< błąd>, Unit>` - w dartz `Unit` to taki `void` czyli nic. Czyli rejestracja i logowanie zwróci nam albo błąd (wcześniej je określaliśmy) albo nic, bo nie potrzebuje od tych metod nic - one mają tylko po stronie firebase zarejestrować lub zalogować usera. Warte zauważenia jest to, że nie przyjmują one adresu email i hasła jako `Stringów` tylko jako nasze `ValueObjecty` definiowane w domain czyli `EmailAddress` i `Password`. Dzięki temu można wykonać na nich `getOrCrash()` jeżeli nie spełniają wymogów. Wylogowanie mówi samo za siebie natomiast getSignerUser to `Option<User>` z dartz czyli łopatologicznie albo `User` albo `null`. Tutaj używamy naszego mappera aby z `Usera` firebase zrobić naszego Usera domenowego.

Huzzah! Infrastructure skończony. Wasza brand new apka dalej pokazuje flutter counter a napaliliśmy kodu aż miło. Dalej przed nami Application i Presentation. Następnym krokiem będzie oczywiście Application bo:

- zdefiniowaliśmy domenę: co? jak? itp.
- zbudowaliśmy implementacje: jak to zrobimy?
- teraz w application zbudujemy tych danych za pomocą narzędzia do zarządzania stanem. W naszym przypadku StateNotifer+riverpod+freezed ale równie dobrze może to być bloc, Provder+ChangeNotifer, Mobx czy coś innego. Te warstwy apki działają niezależnie
- Potem dopiero zbudujemy widoki przekazując do nich dane a Application, a które uzyskaliśmy w Infrastructure. Widzicie jak to się kaskadowo nakłada.
- Potem potestujemy i ustawimy Github Actions.

Ale to potem, bo musze jeszcze postawić jakiegoś bloga pod to i nie wiem czy ktoś to w ogóle będzie czytał.
