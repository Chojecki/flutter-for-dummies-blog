---
title: Flulander cz. 2 / 2
date: "2020-10-04"
description: "Karawana jedzie dalej"
---

W części pierwszej naszego Firebase Authentication, zbudowaliśmy sobie warstwy Domain i Infrastructure czyli Co i jak zrobimy. W części drugiej będzie trochę łatwiej i mnie odtwórczo względem np. linkowanego kuru Resocodera ( ciężko jest zrobić coś inaczej, jeżeli używamy tego samego podejścia i stacku). Teraz jednak zbudujemy nasze widoki, ale przed tym warstwę aplikacji w której zajmiemy się zarządzaniem stanem. I tak jak Matej używa flutter_bloc tak my użyjemy Riverpod+freezed, a to wymusi na nas trochę różnic. Lecimy z koksem.

![Zoolander](https://media4.giphy.com/media/xT77XRlFIooQYQsy6A/200.gif)

#### APPLICATION:

Warstwa aplikacji do w moim odczuci logika, a jeszcze prościej, nasze zarządzanie stanem. Przyjęło się mówić nowoczesnych deklaratywnych frameworkach jak React, SwiftUI czy właśnie Flutter stan naszej aplikacji definiuje jego widoki.

> Narzędzi do zarządzania stanem jest bardzo dużo. Moim jednak zdaniem, najbardziej optymalne będą te, w których stan jest > immutable. Przykładem jest np Cubit/flutter_bloc, Redux czy StateNotifier. StateNotifier jest osobną paczką, ale wchodzi > w skład Riverpod, dlatego będziemy go używać.

##### Stan

Dobra ale po primo to taki stan musimy sobie zdefiniować. W podejściu ChangeNotifier+Provider, często widzę, że ktoś po prostu tworzy zmienną X w klasie ChangeNotifer i potem dostarcza ją do widoków za pomocą np. Selecta.

My nasz stan zdefiniujemy jako Union w osobnym pliku. A więc w folderze /application tworzymy folder “auth” (nazwa naszej domeny biznesowej). Tworzymy AuthState w którym definiujemy 3 stany: początkowy, niezalogowany i zalogowany. Union tworzymy oczywiście za pomocą freezed.

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'auth_state.freezed.dart';

@freezed
abstract class AuthState with _$AuthState {
  const factory AuthState.initial() = Initial;
  const factory AuthState.authenticated() = Authenticated;
  const factory AuthState.unauthenticated() = Unauthenticated;
}
```

Następnym plikiem jaki tworzymy jest `AuthController` i zaczynamy zabawę ponieważ będzie to klasa rozszerzająca `StateNotifer<T>`. Ten typ <T> będzie właśnie typem naszego stanu. StateNotifer daje nam swoją zmienną `state`, która nie oznacza nic innego jak jego aktualny stan typu <T>.

Nasz `AuthController` będzie przyjmował jedną zależność i będzie to `IAuthFacade`. Zwróćcie uwagę, że to klasa abstrakcyjna z naszej domeny a nie implementacja z infrastructure. Dzięki temu `AuthController` jest nie zależny od implementacji ale zgodny z domeną - ma zależność, która wie co ma robić, ale nie wie jak. Teraz jeżeli w przyszłości zrezygnujemy z `Firebase` i użyjemy `Icebase`, to będziemy musieli tylko stworzyć implementację `IAuthFacade` z użyciem Icebase a nasz `AuthController` pozostanie bez zmian i będzie działał.

Dalej w konstruktorze `super` definiujemy jaki będzie początkowy stan `StateNotifiera`. Nasze `<T>` to `<AuthState>` czyli Union freezed który zrobiliśmy przed chwilą. Tak więc na początku pod zmienną `state` kryje się `AuthState.initial()`. W ciele konstruktora, odpalamy sobie jeszcze jedną metodę `authCheckRequest`, po to aby wywołała się ona od razu po użyciu klasy `AuthController` a zrobimy to w widoku z pomocą riverpod.

Metody AuthControllera to:

- sprawdzenie czy user jest zalogowany
- wylogowanie

Zobacz, że wzór wygląda podobnie - za pomocą `_authFacade` używamy zależności, aby zrobić coś po stronie Firebase, a następnie przypisujemy do zmiennej `state` dany stan typu AuthState. `getSignedUser` zwraca Option<User>, więc kożystamy z `fold` paczki `dartz` który łopatologicznie oznacza - pierwszy callback jak null, drugi callback jak coś (w tym przypadku `nasz User`).

```dart
import 'package:riverpod/all.dart';

class AuthController extends StateNotifier<AuthState> {
  AuthController(this._authFacade) : super(const AuthState.initial()) {
    authCheckRequested();
  }

  final IAuthFacade _authFacade;

  Future<void> authCheckRequested() async {
    final userOption = await _authFacade.getSignedInUser();
    state =
        userOption.fold(() => const AuthState.unauthenticated(), (_) => const AuthState.authenticated());
  }

  Future<void> signedOut() async {
    await _authFacade.signOut();
    state = const AuthState.unauthenticated();
  }
}

final authContollerProvider = StateNotifierProvider<AuthController>(
  (ref) {
    final repo = ref.read(firebaseAuthFacadeProvider);
    return AuthController(repo);
  },
);

```

A co to za dziwny `authControllerProvider` pod klasą? Później w widokach, będziemy potrzebować informacji o tych stanach - jednym słowem widoki będą reagować na zmiany stanu i tak jak wcześniej wspomniałem, w takim ChangeNotifierze przewnie ktoś użyłby `Providera` i `Selecta`. W flutter_bloc/cubit, `BlocProvera`, `BlocConumsera` itp. Całe piękno Riverpod polega na tym, że nie musimy się bawić w cały ten syf. Poprostu tworzymy sobie w dowolnym miejscu w aplikacji (u nas w tym samym pliku), `StateNotifieProvider`, który w widokach użyjemy po prostu za pomocą Hooków. Prosto i bez dziwnych builderów, a co najważniejsze, bardzo zoptymalizowane. Riverpod tak samo jak Provider ma dużo tych swoich providerów. `StateNotifierProvider` jak jego nazwa wskazuje, dedykowany jest dla `StateNotifiera` i ma kilka udogodnień w których np. nie musimy wywoływać jego metody za pomocą `context.read()` tylko bezpośrednio ze zmiennej do której zbindowaliśmy StateNotifierProvider czyli można zrobić `zmienna.metoda()` i będzie gitara. W pierwszej części, wspomniałem też, że w Riverpod, providery mogą czytać siebie nawzajem za pomocą `ref`. W naszym Infrastructure i implementacji IAuthFacade, `FirebaseAuthFacade` również stworzyliśmy `Provider` o nazwie `firebaseAuthFacade`. `AuthFormContoller` ma zależność od dowolnej `IAuthFacade`, więc za pomocą `ref` czytamy sobie FirebaseAuthFacade, tworzymy instancje `AuthFormController` i ją zwracamy. Uff mam nadziej, że ma to sens. Jeżeli nie, to spróbuj o tym pomyśleć lub poczytaj dokumentację Riverpoda, bo to jest naprawdę turbo proste w praktyce tylko brzmi zawile.

To w sumie wszystko do zarządzania stanem naszej `auth`. Jednak podobnie jak we flutter_bloc wydzielamy sobie bloki na osobne logiki, tak i tutaj zrobimy sobie jeszcze obsługę stanu formularza (wpisywania emaila i hasła) za pomocą StateNotifiera.

Tworzymy w `/application/auth` subfolder `auth_form` i dalej powtarzamy dokładnie to samo co przed chwilą: Tworzymy stan za pomocą freezed i kontroler do tego stanu, który jest StateNotifierem i gitara. W tym momencie, ktoś pomyśli hah, no kurrrr proste, robię copy/paste i przesyłam numer konta. Niestety nie ma tak pięknie. Owszem, stan zdefiniujemy tak samo z freezed, ale tym razem nie będzie to Union a data class. Czyli zwykła klasa freezed. Tworzymy `auth_form_state.dart`

```dart
import 'package:dartz/dartz.dart';
import 'package:freezed_annotation/freezed_annotation.dart';

part 'auth_form_state.freezed.dart';

@freezed
abstract class AuthFormState with _$AuthFormState {
  const factory AuthFormState({
    @required EmailAddress emailAddress,
    @required Password password,
    @required bool showErrorMessages,
    @required bool isSubmiting,
    @required Option<Either<AuthFailure, Unit>> authFailureOrSuccessOption,
  }) = _AuthFromState;

  factory AuthFormState.initial() => AuthFormState(
      emailAddress: EmailAddress(''),
      password: Password(''),
      showErrorMessages: false,
      isSubmiting: false,
      authFailureOrSuccessOption: none());
}

```

Chodzi o to, że nie będziemy tutaj zwracać innego stanu po zmianie, ale kopię tego samego stanu z jakąś zmianą typu inny email. Brzmi to dziwnie, ale zaraz zobaczymy jak to działa w StateNotiferze. Czyli tworzymy `AuthFromState`, definiujemy jego parametry i jedną fabrykę jako stan początkowy, który posłuży nam jako ten, który przekażemy w `super` StateNotifiera, aby od początku miał jakiś stan - w naszym wypadku czysty formularz. Uwagę może przykuć `authFailureOrSuccessOption`, który jest albo nullem (Option), albo porażką, albo sukcesem (Unit czyli void czyli nic) - chodzi w tym o to, że na próbę zalogowania dostaniemy albo konkretny błąd czemu się nie udało albo info o tym, że jest gitarka.

Dalej tworzymy `auth_form_controller.dart` i może on wyglądać na skomplikowany z uwagi na metodę `_performActionOnFacadeWithEmailAndPassword` i jest to coś co zapożyczyłem z kursu Resocodera, ponieważ robił to samo w swoim Blocu. Jakkolwiek zawile to nie wygląda, to polega na tym, że w nasze logowanie i rejestracja są bardzo podobne. W obu tych rzeczach mamy ten sam EmailAddress i ten sam Password, tak samo je walidujemy i jedyna różnica, to że na koniec z naszej fasady nie wywołujemy `login` tylko `register`. Więc w myśl zasady DRY czyli nie powtarzania tego samego kodu, po prostu do tej metody, przekazujemy metodę fasady. A działa to tak, że walidujemy nasz EmaiAddress i Password, odpalamy przekazany callback (login albo register), ustawiamy stan na `isSubmiting` aby wyświetlić loading i na koniec jak wiemy jaki jest wynik próby logowania/rejestracji zwracamy stan z sukcesem albo porażką. Ustawiamy też `showErrorMessage` na `true`, ponieważ od tego uzależnimy wyświetlanie naszych errorów pod `TextFieldem` - chodzi o to, że jak wiesz `ValueObject` będzie się walidował przy wpisywaniu, więc jeżeli Twój email to dupa@123.pl, to kiedy wpiszesz ‘d’, `EmailAddres` się waliduje i zwróci error -> “Yo, zły format maila”. Więc, aby user nie pomyślał sobie “Ej no, wyluzuj mordeczko, już wpisuje”, to wyświetlamy errory dopiero jak `showErrorMessages` jest `true`.

Zwróć uwagę, że zawsze robimy state = `state.copyWith...`. Tak jak wspomniałem `state` StateNotifiera jest immutable i nie możemy go zmienić a jedynie nadpisać nowym, stąd kopiujemy poprzedni stan jako zupełnie nowy obiekt i przypisujemy go do stanu ze zmianami (np. zmiana emialAddress, która będzie odpalać się na każdą literkę wpisaną w TextField)

```dart
import 'package:dartz/dartz.dart';
import 'package:riverpod/all.dart';

class AuthFormContoller extends StateNotifier<AuthFormState> {
  AuthFormContoller(this._authFacade) : super(AuthFormState.initial());

  final IAuthFacade _authFacade;

  Future<void> emailChanged(String email) async {
    state = state.copyWith(
      emailAddress: EmailAddress(email),
      authFailureOrSuccessOption: none(),
    );
  }

  Future<void> passwordChanged(String password) async {
    state = state.copyWith(
      password: Password(password),
      authFailureOrSuccessOption: none(),
    );
  }

  Future<void> registerWithEmailAndPasswordPressed() async {
    await _performActionOnAuthFacadeWithEmailAndPassword(_authFacade.registerWithEmailAndPassword);
  }

  Future<void> signInWithEmailAndPasswordPressed() async {
    await _performActionOnAuthFacadeWithEmailAndPassword(_authFacade.signInWithEmailAndPassword);
  }

  Future<void> _performActionOnAuthFacadeWithEmailAndPassword(
      Future<Either<AuthFailure, Unit>> Function(
              {@required EmailAddress emailAddress, @required Password password})
          forwardedCall) async {
    Either<AuthFailure, Unit> failureOrSuccess;

    final isEmailValid = state.emailAddress.isValid();
    final isPasswordValid = state.password.isValid();

    if (isEmailValid && isPasswordValid) {
      state = state.copyWith(isSubmiting: true, authFailureOrSuccessOption: none());

      failureOrSuccess = await forwardedCall(emailAddress: state.emailAddress, password: state.password);
    }

    state = state.copyWith(
      isSubmiting: false,
      showErrorMessages: true,
      authFailureOrSuccessOption: optionOf(failureOrSuccess),
    );
  }
}

final authFormContollerProvider = StateNotifierProvider<AuthFormContoller>(
  (ref) {
    final repo = ref.read(firebaseAuthFacadeProvider);
    return AuthFormContoller(repo);
  },
);

```

Yeeey, mamy naszą warstwę aplikacji! Czyli

- Ustaliliśmy jakie mogą być stany domeny autoryzacji
- Ustaliliśmy jak nimi zarządzać
- Stworzyliśmy providery aby dostarczać kontrolery do widoków

#### Presentation

I stało się, zaczynamy prace z… Flutterem. Może nie zauważyliście, ale przez cały ten czas, praktycznie nie używaliśmy Fluttera. Pisaliśmy w Darcie. Dopiero teraz, w warstwie widoku zaczynamy “bawić się w paddingi”. No to jazda.

Zanim pójdzie do folderu presentation, w katalogu głównym mamy `main.dart` :

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/all.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(const ProviderScope(child: AppWidget()));
}

```

Tutaj ważne, aby naszą apkę opakować w `ProviderScope` - jest to nam potrzebne, aby używać Riverpoda.

Następne co, to tworzymy nasz `router.dart`. Użyjemy super paczki o nazwie `auto_route` i jest to nic innego jak kolejny generator kodu, który stworzy nam nawigację w oparciu o podejście `onGeneratedRoutes`.

dsk
Flutter 1.22 pozwala nam używać nowego API do nawigacji, ale Navigator 1.0, wciąż będzie działać.

Tak więc nasz `router.dart` wygląda tak:

```dart
import 'package:auto_route/auto_route_annotations.dart';

@MaterialAutoRouter(
  routes: <AutoRoute>[
    MaterialRoute(page: SplashScreen, initial: true),
    MaterialRoute(page: AuthFormScreen),
    MaterialRoute(page: HomeScreen),
  ],
)
class $Router {}
```

W tej chwili sypie błędami bo nie mamy tych widoków (Splash, AuthFrom, Home), więc na tę chwilę można je po prostu stworzyć jako dummy widgety, zwracające null. Tworzymy je kolejno w subfolderach `/presentation/pages/<nazwa_domeny np auth>/plik.dart`

Po odpaleniu `build_runnera`, mamy naszą nawigację i inital route jako `SplashScreen`.

Czyli mamy folder presentation a w nim folder pages. W samym /presentation tworzymy nasze `app.dart`, które wywołaliśmy w `main.dart`. Wygląda to tak:

```dart
import 'package:auto_route/auto_route.dart' as auto_router;
import 'package:flutter/material.dart' hide Router;

class AppWidget extends StatelessWidget {
  const AppWidget({Key key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Nazwa',
      theme: MainTheme.theme,
      builder: auto_router.ExtendedNavigator.builder(
        router: Router(),
      ),
    );
  }
}

```

Jakiś tam tytuł, jakieś tam theme (już nie będę tego pokazywał) i builder. Builder jest ciekawy, bo za pomocą `ExtendedNavigatora` tworzymy nasz `Router`. Wszystko to pochodzi z `auto_route` i właśnie dlatego importujemy tę paczkę jako `as auto_route` ponieważ Flutter 1.22 ma nowe API do nawigacji i występuje konflikt nazw.

Nasz pierwszy ekran to `SplashScreen` i widać w nim całe piękno Riverpod + StateNotifier + freezed. Zadaniem naszego Splash Screena jest wyświetlenie czegoś początkowego jak loading i następnie podmiana drzewa w zależności od stanu autoryzacji. Zwracając uwagę, że nie mamy tam żadnych builderów, providerów, ifów itp. zobaczcie sami

```dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:hooks_riverpod/all.dart';

class SplashScreen extends HookWidget {
  const SplashScreen({Key key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final authState = useProvider(authContollerProvider.state);
    return AnimatedSwitcher(
      duration: const Duration(milliseconds: 700),
      child: authState.map(
          initial: (_) => Container(
                color: Colors.green,
              ),
          authenticated: (_) => const HomeScreen(),
          unauthenticated: (_) => const AuthFormScreen()),
    );
  }
}
```

Pierwsze co, to nasz `SplashScreen` rozszerza `HookWidget` - czyli mamy hooki (trzeba zainstalować paczki hooks_riverpod, ale tak jak wspominałem w pierwszej części, omijam takie podstawowe założenia celowo, uznając, że każdy kto to czyta, dobrze wie co trzeba zrobić). Dyskusja na temat Hooków jest długa i szeroka. Nie będę w nią wchodził i powiem tylko, że można używać Riverpoda bez hooków, ale z hookami jest fajniej. Dzięki temu w metodzie `build` widgeta możemy użyć hooka `useProvider`, który pozwoli nam do zmiennej `authState` zbindować stan naszego `AuthControllera` (pamiętacie, że `authControllerProvder` tworzyliśmy za pomocą StateNotfierProvidera).

Dalej do akcji wkracza freezed! Stan naszego `AuthControllera` to `AuthState` czyli Union stworzony przez freezed. Dzięki temu możemy na zmiennej `authState` odpalić metodę `map` i “mappować” sobie stan w zależności od którego wyświetlimy dany screen. Teraz jeżeli stan się zmieni, to Riverpod poinformuje o tym Splash i podmieni nam drzewo.

Przechodzimy więc do naszego `AuthFormScreen`. Jego zadaniem jest wyświetlić formularz, ale także zareagować na zmianę stanu i ponownie sprawdzić czy user jest zalogowany, po próbie logowania/rejestracji. W tym celu bindujemy sobie 3 zmienne za pomocą Hooka `useProvider` tak samo jak w `SplashScreenie`. Zmienne te to state, authController oraz authFromController. Kontrolery będą nam potrzebne do odpalania ich metod. Dalej to co zwracamy to `ProviderListener`, który działa tak, że dostaje provider i jeżeli ten provider się zmieni, to możemy reagować na zmianę. Jego childem będzie już `AuthForm` czyli widget z formularzem, który zaraz sobie zrobimy i do którego przekażemy zbindowane zmienne.

```dart
class AuthFormScreen extends HookWidget {
  const AuthFormScreen({Key key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final state = useProvider(authFormContollerProvider.state);
    final authController = useProvider(authContollerProvider);
    final authFormController = useProvider(authFormContollerProvider);

    return ProviderListener<AuthFormState>(
      provider: authFormContollerProvider.state,
      onChange: (context, state) {
        state.authFailureOrSuccessOption.fold(
          () {},
          (either) => either.fold(
            (failure) {
              FlushbarHelper.createError(
                message: failure.map(
                  serverError: (_) => 'Server error',
                  emailAlreadyInUse: (_) => 'Email already in use',
                  invalidEmailAndPasswordCombination: (_) => 'Invalid email and password combination',
                ),
              ).show(context);
            },
            (_) {
              authController.authCheckRequested();
            },
          ),
        );
      },
      child:  AuthForm(
            state: state,
            authFormController: authFormController,
    );
  }
}
```

W `ProviderListener` nasłuchujemy na zmiany stanu w AuthFormControllerze. Jak pamiętacie, jego stan już nie jest Unionem, a klasą freezed i ma parametr `authFailureOrSuccessOption` który jest typu Either błąd albo i nic, a na początku jest nullem. Więc teraz słuchamy czy jego stan się zmienił i jeżeli `authFailureOrSuccessOption` to błąd - wyświetlamy ten błąd. A jeżeli jest niczym to jeszcze raz pytamy o zalogowanego usera poprzez `authCheckRequested`. Jeżeli tym razem metoda ta zwróci `User` to stan `AuthControllera` zmieni się z unauthenticated na authenticated, a to przebuduje drzewo z `SplashScreen`a i podmieni AuthFromScreen na HomeScreen.

Pozostaje nam sam `AuthForm` wigdet. Przekazaliśmy mu:

- stan: do wyświetlania go
- authFormController: do odpalania metod (zmień stan maila, hasła, zaloguj, zarejestruj)

```dart
class AuthForm extends StatelessWidget {
  const AuthForm(
      {Key key,
      @required this.state,
      @required this.authFormController,})
      : super(key: key);

  final AuthFormState state;
  final AuthFormContoller authFormController;

  @override
  Widget build(BuildContext context) {
    return Form(
      key: const Key('__authForm__'),
      autovalidateMode:
          state.showErrorMessages ? AutovalidateMode.onUserInteraction : AutovalidateMode.disabled,
      child: Center(
        child: Container(
          padding: const EdgeInsets.all(16.0),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.end,
            children: [
              const SizedBox(height: 8),
              TextFormField(
                  key: const Key('__email__'),
                  decoration: const InputDecoration(
                    floatingLabelBehavior: FloatingLabelBehavior.never,
                    labelText: 'Email',
                  ),
                  autocorrect: false,
                  onChanged: (value) => authFormController.emailChanged(value),
                  validator: (_) {
                    return state.emailAddress.value.fold(
                        (f) => f.maybeMap(orElse: () => null, invalidEmail: (_) => 'Invalid Email'),
                        (_) => null);
                  }),
              const SizedBox(height: 8),
              TextFormField(
                  key: const Key('__password__'),
                  decoration: const InputDecoration(
                      labelText: 'Password', floatingLabelBehavior: FloatingLabelBehavior.never),
                  autocorrect: false,
                  onChanged: (value) => authFormController.passwordChanged(value),
                  validator: (_) {
                    return state.password.value.fold(
                        (f) => f.maybeMap(orElse: () => null, shortPassword: (_) => 'Password to short'),
                        (_) => null);
                  }),
              const SizedBox(height: 8),
              TextButton(
                onPressed: () => authFormController.signInWithEmailAndPasswordPressed()
                child: Text('Login')
              ),
              TextButton(
                onPressed: () => authFormController.registerWithEmailAndPasswordPressed(),
                child: Text('Register')
              ),
              if (state.isSubmiting) ...[
                const SizedBox(height: 8),
                const LinearProgressIndicator(),
              ]
            ],
          ),
        ),
      ),
    );
  }
}
```

Na uwagę zasługują tutaj:

- ze `state` bierzemy obecny stan i używamy go do walidacji. Zarówno EmailAddress jak i Password to `ValueObject` więc możemy przekazać “wynik” jego `fold` do validatora. Możemy też w zależności od `isSubmiting` wyświetlić loading indicator.
- authFromController posłuży nam do odpalania metod w buttonach

#### I nagle…

![Zoolander pose](https://thumbs.gfycat.com/ColossalAggravatingCooter-small.gif)

Mamy zakończoną domenę naszej autoryzacji. Możemy logować i rejestrować użytkowników i to w super bezpieczny sposób. Podsumujmy:

- W Domain zdefiniowaliśmy sobie co chcemy zrobić
- W Infrastructure, że użyjemy Firebase i jak to zrobimy
- W Application jakiego narzędzia do zarządzania stanem użyjemy i jak będzie wyglądał stan naszej apki
- W Presentation przelaliśmy to na widoki

Moim zdaniem to super podejście do budowania i rozwijania apki. Daje nam dużą elastyczność nawet w przypadku tak dużej ingerencji jak zmiana libki do zarządzania stanem czy zmiana backendu nawet kiedy appka będzie mocno rozbudowana (edytujemy dane foldery). W dodatku fajnie można sobie podzielić pracę np. “Ja zrobię nowy ficzer a Ty widoki” i bardzo mało prawdopodobne, że podczas mergowania, będziecie mieli konflikty nawet jeżeli zupełnie przerobicie swoje części, bo będziecie tak naprawdę pracować w innych folderach.

Jeżeli spodobało wam się to podejście, to zachęcam z całego serca do obejrzenia kursu DDD by ResoCoder [tutaj](https://www.youtube.com/playlist?list=PLB6lc7nQ1n4iS5p-IezFFgqP6YvAJy84U). Ja tylko otarłem się o temat, a u Mateja zobaczycie kawał dobrego kodu w całej apce.

### Bonus (RPK)

#### Testowanie

Mam nadzieje, że nikogo nie muszę namawiać do testowania. Ja myślę zawsze o testach jak o “Sejwie” w grze RPG - kiedy mam na coś napisany test, to wiem, że jeżeli ktoś (lub ja) zacznie grzebać w kodzie, to dopóki test przechodzi, wszystko co zrobiłem do tej pory jest jak miało być. Nawet przykład z tego projektu. Flutter 1.22 informuje nas, że w widgecie `Form` już nie powinniśmy używać `autovalidate`, które miałem ustawione na `state.showErrorMessage`. Więc luźno zmieniłem `autovalidate` na `AutoValidateMode.onUserInteraction` i byłem zadowolony. Puszczam test i… hola hola piękny chłopcze. Testy failują. Patrzę co jest nie tak i faktycznie w takim ustawieniu, pola email i password, walidować się będą od razu a nie po próbie zalogowania/zarejestrowania. Więc szybka poprawka na to co jest teraz i działa jak powinno. Aż strach pomyśleć ile takich głupot może nam przejść obok nosa bez testów.

Ale dobra, to oto jak wygląda wyżej opisany test case. Czyli po odpaleniu appki i niezalogowanym userze, mamy dostać authform. Ma nie być żadnych errorów. User wpisuje zły format emaila i za któtkie hasło i klika button. User ma dostać errory, że zły format i za krótkie hasło.

```dart
class MockFirebaseAuthFacade extends Mock implements IAuthFacade {}

void main() {
  final mockAuthFacade = MockFirebaseAuthFacade();

testWidgets(
    'Display errors under the text fields in AuthScreen',
    (WidgetTester tester) async {
      when(mockAuthFacade.getSignedInUser()).thenAnswer(
        (_) => Future.value(
          optionOf(null),
        ),
      );

      await tester.pumpWidget(
        MaterialApp(
          home: ProviderScope(
            overrides: [
              firebaseAuthFacadeProvider.overrideWithProvider(Provider((ref) => mockAuthFacade))
            ],
            child: const SplashScreen(),
          ),
        ),
      );

      final email = find.widgetWithText(TextFormField, 'Email');
      final password = find.widgetWithText(TextFormField, 'Password');
      final signInButton = find.widgetWithText(TextButton, 'Login');

      await tester.enterText(email, '123');
      await tester.enterText(password, '12');
      await tester.pump();

      expect(find.text('Invalid Email'), findsNothing);
      expect(find.text('Password to short'), findsNothing);

      await tester.tap(signInButton);
      await tester.pumpAndSettle();

      expect(find.text('Invalid Email'), findsOneWidget);
      expect(find.text('Password to short'), findsOneWidget);
    },
  );
}
```

Ale dobra, to oto jak wygląda wyżej opisany test case. Czyli po odpaleniu appki i niezalogowanym userze, mamy dostać authform. Ma nie być żadnych errorów. User wpisuje zły format emaila i za krótkie hasło i klika button. User ma dostać errory, że zły format i za krótkie hasło. tera

I teraz czemu to jest takie super? Pamiętacie, że w domain mamy `IAuthFacade` a w infrastructure jego implementacje `FirebaseAuthFacase`? To znów mamy szansę zobaczyć piękno naszej architektury, ponieważ za pomocą paczki `mockito` (chyba wszyscy znają i kochają), tworzymy mockową klasę `MockFirebaseAuthFacade`, która implementuje nasz `IAuthFacade` czyli naszego Pana Kierownika co wie co ma być zrobione ale nie wie jak.

I Teraz `mockito` pozwala nam zdefiniować co się stanie jeżeli wykonamy jakiś plan naszego Pana Kierownika za pomocą `when`. Czyli `IAuthFacade` narzucał, że jego implementacja ma mieć metodę `getSignedUser`. My implementowaliśmy to za pomocą `FirebaseAuthFacade` i zwracaliśmy Usera lub null. To teraz za pomocą `when` mówimy aby przy użyciu `MockFirebaseAuthFacade` kiedy wywołamy `getSignerUser` mamy dostać nulla.

Pamiętamy, że ta metoda wywoła się na początku apki i jeżeli zwróci null to dostaniemy AuthScreen. Wszystko spoko, tylko my w apce mamy `firebaseAuthFacadeProvider` który wywołuje nam tę metodę, a Provider ten korzysta z `FirebaseAuthFacade` a nie naszego mocka. Więc tutaj ponownie mamy super narzędzie od Riverpoda! I jest to `ProviderScope`. Ten sam w który zapakowaliśmy naszą appkę w `main.dart`. Używając go tutaj jeszcze raz, możemy na dowolnym Providerze wywołać `overrideWithProvider` i w ten sposób wszędzie tam, gdzie w naszej apce używamy tego Providera, to podczas testów “użyje się” Provider, który przekażemy w tym callbacku, a będzie to nasz świeżutki `MockFirebaseAuthFacade`, któremu już powiedzieliśmy, że przy pobieraniu Usera ma dać nam nulla. Uff mam nadzieje, że chociaż w miarę sensownie to opisałem :/

I to tyle.
