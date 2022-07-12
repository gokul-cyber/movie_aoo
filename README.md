
By logging in, we mean that we need a session Id. This session ID will be further used to get the account details, rate a movie, delete a rating, add a movie to the watchlist, and bring the watchlist. You can also delete the session ID.

There are 3 steps to get the Session ID -

Get a request token by /authentication/token/new. This will give a request token.
Validate the username and password for your account by making POST call /authentication/token/validate_with_login with username, password, and request token received in step 1, in the request body to get another request token. If you receive the token, that means the username and password are correct and you can proceed with creating the session. But, if the username and password are invalid, you won’t get the request token and won’t be able to get the session id.
With the request token received in the 2nd step, call this POST API /authentication/session/new to get the session id.
I hope these steps are crystal clear and now we can start with writing calls to these API by creating a data source and adding post method to the ApiClient.

Generic POST Call
Open api_client.dart and create a post method

//1
dynamic post(String path, {Map<dynamic, dynamic> params}) async {
  //2
  final response = await _client.post(
    getPath(path, null),
    //3
    body: jsonEncode(params),
    //4
    headers: {
      'Content-Type': 'application/json',
    },
  );
  //5
  if (response.statusCode == 200) {
    return json.decode(response.body);
  }
  //6 
  else if (response.statusCode == 401) {
    throw UnauthorisedException();
  } 
  //7
  else {
    throw Exception(response.reasonPhrase);
  }
}
Create a post method, that takes path and params. Here, params are the map that is a request payload that will be passed in the body.
Call the _client.post method, with the path. We are using the same getPath() with params being null because in our params we will not have any query parameters.
Next, add a body to the request with the params that are passed in the method.
Keep the headers for the request, same as that in the get call.
If the response is a success, we parse the JSON response.
If the response is 401, i.e. unauthorized, we throw an UnauthorisedException, which we have created. This is a simple field less class as of now which implements the Exception class.
Last, we will handle any type of exception that can be thrown by POST API.
Data Source
Now, we are creating a new data source that will deal with authentication-related API calls. Create a new file in datasources folder. As explained before, we will have 3 calls in sequence to get the session id. So, create the 3 methods.

abstract class AuthenticationRemoteDataSource {
  //1
  Future<RequestTokenModel> getRequestToken();
  //2
  Future<RequestTokenModel> validateWithLogin(Map<String, dynamic> requestBody);
  //3
  Future<String> createSession(Map<String, dynamic> requestBody);
}
Get the request token.
Validate the username and password.
Create a session.
The response of the first 2 APIs is the same as this -

{
  "success": true,
  "expires_at": "2021-03-13 05:48:07 UTC",
  "request_token": "5e29eca2e02b5adaf9a659e41bb4ca1bd6bcc0fd"
}
So, let’s create a model to store this token.

class RequestTokenModel {
  //1
  final bool success;
  final String requestToken;
  final String expiresAt;
  RequestTokenModel({
    this.success,
    this.requestToken,
    this.expiresAt,
  });
  //2
  factory RequestTokenModel.fromJson(Map<String, dynamic> json) {
    return RequestTokenModel(
      success: json['success'],
      requestToken: json['request_token'],
      expiresAt: json['expires_at'],
    );
  }
  //3
  Map<String, dynamic> toJson() => {
        'request_token': requestToken,
      };
}
As per the JSON response, we will have 3 fields. Create a constructor for the class.
Create the factory method fromJson that will parse the JSON response to the model.
Create a toJson() to use when sending this request token in the second and third requests.
We don’t need corresponding RequestTokenEntity for RequestTokenModel, because the domain doesn't need to worry about how the session is created. Domain only needs a session id and getting a request token is one of the processes to get session id.

Now, let’s implement the data source methods. In the same file where the abstract data source class is present, create an implementation class of that.

class AuthenticationRemoteDataSourceImpl
    extends AuthenticationRemoteDataSource {
  //1
  final ApiClient _client;
  AuthenticationRemoteDataSourceImpl(this._client);
  @override
  Future<RequestTokenModel> getRequestToken() async {
    //2
    final response = await _client.get('authentication/token/new');
    print(response);
    final requestTokenModel = RequestTokenModel.fromJson(response);
    return requestTokenModel;
  }
  @override
  Future<RequestTokenModel> validateWithLogin(
      Map<String, dynamic> requestBody) async {
    //3
    final response = await _client.post(
      'authentication/token/validate_with_login',
      params: requestBody,
    );
    print(response);
    return RequestTokenModel.fromJson(response);
  }
  @override
  Future<String> createSession(Map<String, dynamic> requestBody) async {
    //4
    final response = await _client.post(
      'authentication/session/new',
      params: requestBody,
    );
    print(response);
    return response['success'] ? response['session_id'] : null;
  }
}
We will need an instance of ApiClient to mak get API and post API calls in this data source.
To get the request token, authentication/token/new is the path and we will parse the response into RequestTokenModel.
Validate with login will have a requestBody as it will consist of the username, password, and request token. This is a POST call and we will pass in params to the post(). Parse the response into RequestTokenModel again.
Lastly, we have createSession POST API call that takes in request token as well. If we skip step 2 validate with login and try to get the session, we will get Session denied error where success will be false. So, before returning we are checking the success. If true, we return true else null. We will handle this null value in the repository layer.
Repository
We have created methods in data source, now let’s make repository methods. Create a new abstract repository class in domain layer.

abstract class AuthenticationRepository {
  //1
  Future<Either<AppError, bool>> loginUser(Map<String, dynamic> params);
  //2
  Future<Either<AppError, void>> logoutUser();
}
We will have only 2 methods here as we have majorly login and logout as of now. The loginUser() takes in request body params username and password. The return type will be either error or a boolean success, which will mostly be true.
We will also see logout functionality later in the article, so let’s just create this method as well which returns nothing.
Let’s implement the most important part of this article. We will implement these 2 methods in the implementation class in the data layer as usual. This class will take an instance of AuthenticationDataSource in the constructor as we do in other repositories. Implement all the methods and create a private method that will fetch the request token.

//1
Future<Either<AppError, RequestTokenModel>> _getRequestToken() async {
  //2
  try {
    final response = await _authenticationRemoteDataSource.getRequestToken();
    return Right(response);
  } 
  //3
  on SocketException {
    return Left(AppError(AppErrorType.network));
  } 
  //4
  on Exception {
    return Left(AppError(AppErrorType.api));
  }
}
//5
@override
Future<Either<AppError, bool>> loginUser(Map<String, dynamic> body) async {
  final requestTokenEitherResponse = await _getRequestToken();
  
}
We kept this method private because this is an intermediate step. This will return either the app error or request token model.
In the try block, you will call the getRequestToken() on the data source. If everything goes well, we return the Right object.
If there is a network connection failure, we return with the Left object of error type as a network as we did in other repositories.
Similarly, if something has failed in the API itself then we return with an API type error.
In this loginUser() call this method now to get the request token.
Next, you’ll work on step 2, and step 3 to bring the session id that I explained at the start.

@override
Future<Either<AppError, bool>> loginUser(Map<String, dynamic> body) async {
  final requestTokenEitherResponse = await _getRequestToken();
  //1
  final token1 =
      requestTokenEitherResponse.getOrElse(() => null)?.requestToken ?? '';
  //2
  try {
    //3
    body.putIfAbsent('request_token', () => token1);
    //4
    final validateWithLoginToken =
        await _authenticationRemoteDataSource.validateWithLogin(body);
    //5
    final sessionId = await _authenticationRemoteDataSource
        .createSession(validateWithLoginToken.toJson());
    //6
    print(sessionId);
    //7
    return Right(true);
  } on SocketException {
    return Left(AppError(AppErrorType.network));
  } 
  //8
  on UnauthorisedException {
    return Left(AppError(AppErrorType.unauthorised));
  } on Exception {
    return Left(AppError(AppErrorType.api));
  }
}
Once you get the request token model, you’ll fetch the right value from either response by using getOrElse(). we should supply the default value in case the response contains the Left object instead of Right. The default value we keep as null and then we try to invoke the requestToken from it. If it's null we return with an empty string.
Now, that we have to make a second data source call, add a try block.
The body passed to this method contains username and password, but we need to add request_token to the body for the second call.
Call the second API, validateWithLogin with the body. If the request token was empty from the first call this API call still returns success but with an empty request token.
Call the last API to create a session with the latest request token received in 2nd call. If we receive an empty request token in the 2nd call, this API will give the success value as false in the response, and then we are fetching session id in the _authenticationRemoteDataSource.createSession() based on this value. Let's just assume that we always get the sessionId. We will handle the error just in a moment after this code block.
Let’s just print the session id for debugging purposes.
Return the Right response with true value.
The session id hasn’t expired over days for me. But it can, we don’t know that. So, we will keep this session id in our local database to make calls like Rate a movie, bring rated movies, add to watchlist and bring the watchlist. For all the APIs that require session id, we will need this. So, let’s keep it in the hive DB.

Authentication Local Data Source
Create a new local data source — AuthenticationLocalDataSource.

//1
abstract class AuthenticationLocalDataSource {
  Future<void> saveSessionId(String sessionId);
}
class AuthenticationLocalDataSourceImpl extends AuthenticationLocalDataSource {
  @override
  Future<void> saveSessionId(String sessionId) async {
    //2
    final authenticationBox = await Hive.openBox('authenticationBox');
    //3
    return await authenticationBox.put('session_id', sessionId);
  }
}
For now, we only need one method that will save the session id in DB.
Implement this method in the implementation class. Open the box.
Insert the sessionId in the box with session_id as key.
Let’s get back to the repository where we will now call this method to save session id.

//1
final AuthenticationLocalDataSource _authenticationLocalDataSource;
AuthenticationRepositoryImpl(
  this._authenticationRemoteDataSource,
  this._authenticationLocalDataSource,
);
final sessionId = await _authenticationRemoteDataSource
    .createSession(validateWithLoginToken.toJson());
//2
if (sessionId != null) {
  await _authenticationLocalDataSource.saveSessionId(sessionId);
  return Right(true);
}
//3
return Left(AppError(AppErrorType.sessionDenied));
Declare the local data source as well in the repository.
In the loginUser(), after getting the session id to save it to the local database if it is not null. It's a successful operation when the session is not null, so we can return the Right object.
In default conditions, we will return the Left object because the only failing condition is that session was denied.
Login Use Case
After the repository, we will move to the usecase. Create a new usecase in the domain layer

//1
class LoginUser extends UseCase<bool, LoginRequestParams> {
  //2
  final AuthenticationRepository _authenticationRepository;
  LoginUser(this._authenticationRepository);
  //3
  @override
  Future<Either<AppError, bool>> call(LoginRequestParams params) async =>
      _authenticationRepository.loginUser(params.toJson());
}
This usecase will take in LoginRequestParams which we will create in the next code block. This is just a data holder for username and password.
Next, you need AuthenticationRepository to call the loginUser().
In the call(), call the loginUser() by invoking toJson() on the LoginRequestParams object, because we have to send the request payload in JSON format.
Now, create the data holder as well in the entities folder.

class LoginRequestParams {
  final String userName;
  final String password;
  LoginRequestParams({
    @required this.userName,
    @required this.password,
  });
  Map<String, dynamic> toJson() => {
        'username': userName,
        'password': password,
      };
}
This data holder class takes in username and password and toJson() to convert the object to JSON format.

Let’s now move to UI but before that, if you haven’t subscribed to the channel yet, please help the channel by subscribing to it. You can also share these articles with your friends, teammates, and anyone who is learning Flutter. Hit the Like button if you are liking them.

UI
Since we don’t have a signup API, I have already created a basic login screen with 2 fields and 1 button. We will also have validation messages when there is an error from the API call. Here is how the UI will look like -

Nothing much to explain in the UI, so I will quickly go through it. By default, we will now open the login screen so change the initial route to the login screen and create another route for the home screen. Open routes.dart and make the changes.

RouteList.initial: (context) => LoginScreen(),
RouteList.home: (context) => HomeScreen(),
New Journey
Let’s create a new journey, by adding a new folder login. Create a new file login_screen.dart

//1
class LoginScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    //2
    return Scaffold(
      resizeToAvoidBottomInset: false,
      body: Center(
        child: Column(
          children: [
            //3
            Padding(
              padding: EdgeInsets.only(top: Sizes.dimen_32.h),
              child: Logo(height: Sizes.dimen_12.h),
            ),
            //4
            LoginForm(),
          ],
        ),
      ),
    );
  }
}
The screen will be Stateless.
Return a Sacffold with resizeToAvoidBottomInset set to false, this will not resize the screen when we open the keyboard.
Use the Logo widget with some height and give some top padding to it. We will have more items below this logo, so use Column and make everything in the center.
In LoginForm we will add 2 text fields, 1 button, and an error message placeholder.
Before we go with UI, I have already added the translation of strings in en.json, es.json, and translation constants, because let’s focus on the main things for this article. You can anytime refer to the source code.

Let’s create the LoginForm now. Create another file in the login folder

//1
class LoginForm extends StatefulWidget {
  @override
  _LoginFormState createState() => _LoginFormState();
}
class _LoginFormState extends State<LoginForm> {
  //2
  TextEditingController _userNameController, _passwordController;
  //3
  @override
  void initState() {
    super.initState();
    _userNameController = TextEditingController();
    _passwordController = TextEditingController();
  }
  @override
  void dispose() {
    _userNameController?.dispose();
    _passwordController?.dispose();
    super.dispose();
  }
  //4
  @override
  Widget build(BuildContext context) {
    return SingleChildScrollView(
      child: Padding(
        padding: EdgeInsets.symmetric(
          horizontal: Sizes.dimen_32.w,
          vertical: Sizes.dimen_24.h,
        ),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            //5
            Padding(
              padding: EdgeInsets.only(bottom: Sizes.dimen_8.h),
              child: Text(
                TranslationConstants.loginToMovieApp.t(context),
                textAlign: TextAlign.center,
                style: Theme.of(context).textTheme.headline5,
              ),
            ),
            //6
            LabelFieldWidget(
              label: TranslationConstants.username.t(context),
              hintText: TranslationConstants.enterTMDbUsername.t(context),,
              controller: _userNameController,
            ),
            //7
            LabelFieldWidget(
              label: TranslationConstants.password.t(context),,
              hintText: TranslationConstants.enterPassword.t(context),,
              controller: _passwordController,
              isPasswordField: true,
            ),
          ],
        ),
      ),
    );
  }
}
Create a Stateful widget.
Declare 2 controllers for each of the fields, username, and password.
Initialize the controllers in the initState() and dispose them in the dispose().
Now, in the build(), use SingleChildScrollView to support very small size screens. Give some vertical and horizontal padding and use Column to put widgets one below each other.
Give a simple message for the user to use TMDb credentials.
We will create the LabelFieldWidget just after this. This widget will take in label, hint text, controller to capture user text.
Below the username field we will have a password field that will also hide the password with a special character, so pass that flag to the widget.
Let’s create the LabelFieldWidget in a separate file.

class LabelFieldWidget extends StatelessWidget {
  final String label;
  final String hintText;
  final bool isPasswordField;
  final TextEditingController controller;
  final UnderlineInputBorder _enabledBorder = const UnderlineInputBorder(
    borderSide: BorderSide(
      color: Colors.grey,
    ),
  );
  final UnderlineInputBorder _focusedBorder = const UnderlineInputBorder(
    borderSide: BorderSide(
      color: Colors.white,
    ),
  );
  const LabelFieldWidget({
    Key key,
    @required this.label,
    @required this.hintText,
    @required this.controller,
    this.isPasswordField = false,
  }) : super(key: key);
  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: EdgeInsets.symmetric(vertical: Sizes.dimen_8.h),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(
            label.toUpperCase(),
            style: Theme.of(context).textTheme.headline6,
            textAlign: TextAlign.start,
          ),
          TextField(
            obscureText: isPasswordField,
            obscuringCharacter: '*',
            controller: controller,
            style: Theme.of(context).textTheme.headline6,
            decoration: InputDecoration(
              hintText: hintText,
              hintStyle: Theme.of(context).textTheme.greySubtitle1,
              focusedBorder: _focusedBorder,
              enabledBorder: _enabledBorder,
            ),
          ),
        ],
      ),
    );
  }
}
Not explaining step by step, but this UI will have a column to have a label and field in the vertical direction. We are using two borders — focused and enabled border. For the password field, we are using a special character.

Let’s add a button now below the fields. I have modified the button with small animation, that will change colors when the isEnabled flag is passed as true or false. let's use that button.

Button(
  onPressed: () {},
  text: TranslationConstants.signIn,
  isEnabled: false,
),
Let’s make this button enabled or disabled when the username and password fields are not empty.

//1
bool enableSignIn = false;
@override
void initState() {
  super.initState();
  _userNameController = TextEditingController();
  _passwordController = TextEditingController();
  //2
  _userNameController.addListener(() {
    setState(() {
      enableSignIn = _userNameController.text.isNotEmpty &&
          _passwordController.text.isNotEmpty;
    });
  });
  _passwordController.addListener(() {
    setState(() {
      enableSignIn = _userNameController.text.isNotEmpty &&
          _passwordController.text.isNotEmpty;
    });
  });
}
//3
Button(
  onPressed: enableSignIn ? () {} : null,
  text: TranslationConstants.signIn,
  isEnabled: enableSignIn,
),
We will declare this bool flag, which will be used in the button and will change its value when both the fields have some text.
Add the listener to both the controllers and call setState() whenever we receive a callback. Whenever both fields are no empty, we will make this field true.
Now use the enableSignIn in isEnabled field. Restart the app and enter some text in both fields. You'll see the button getting enabled when both fields are not empty.
BLoC
Let’s work on Bloc now to handle the signin tap events and some intermediate states when there is an error. Create a new bloc in the blocs folder.

In login_event.dart file, create 2 events

//1
class LoginInitiateEvent extends LoginEvent {
  final String username, password;
  LoginInitiateEvent(this.username, this.password);
  @override
  List<Object> get props => [username, password];
}
//2
class LogoutEvent extends LoginEvent {}
You will create LoginInitiateEvent that will be dispatched when the sign in button is pressed. This will take in username and password as parameters to make the API call in the bloc.
You will also have LogoutEvent that will be dispatched when we want to logout.
In login_state.dart file, create 3 more states

//1
class LoginSuccess extends LoginState {}
//2
class LogoutSuccess extends LoginState {}
//3
class LoginError extends LoginState {
  final String message;
  LoginError(this.message);
  @override
  List<Object> get props => [message];
}
The LoginSuccess will be emitted when we can create the session for the user.
The LogoutSuccess will be emitted when we can log the user out.
The LoginError will be emitted when there is an error while logging the user. We will have an error message to display in this case.
In login_bloc.dart, handle the events now.

//1
final LoginUser loginUser;
LoginBloc({
  @required this.loginUser,
}) : super(LoginInitial());
//2
if (event is LoginInitiateEvent) {
  final Either<AppError, bool> eitherResponse = await loginUser(
    LoginRequestParams(
      userName: event.username,
      password: event.password,
    ),
  );
  yield eitherResponse.fold(
    //4
    (l) {
      var message = getErrorMessage(l.appErrorType);
      return LoginError(message);
    },
    //3
    (r) => LoginSuccess(),
  );
}
//5
String getErrorMessage(AppErrorType appErrorType) {
  switch (appErrorType) {
    case AppErrorType.network:
      return TranslationConstants.noNetwork;
    case AppErrorType.api:
    case AppErrorType.database:
      return TranslationConstants.somethingWentWrong;
    case AppErrorType.sessionDenied:
      return TranslationConstants.sessionDenied;
    default:
      return TranslationConstants.wrongUsernamePassword;
  }
}
Declare the loginUser usecase in the bloc and make it required in the constructor.
If the event is LoginInitiateEvent, you'll hit the loginUser usecase with username and password from the event.
Now get the left and right values from either response and yield LoginSuccess for the right value.
If we get a Left object, we will create an appropriate message and yield it in the LoginError state.
Create a method that returns an appropriate message as per the AppErrorType.
We have created a data source, repositories, usecase, and the bloc. So, we should add the initializations in the get_it.dart file, as we have done in other features.

Let’s use this bloc in the proper place. Open movie_app.dart and add the bloc together with LanguageBloc.

//1
LoginBloc _loginBloc;
//2
_loginBloc = getItInstance<LoginBloc>();
//3
_loginBloc?.close();
//4
MultiBlocProvider(
  providers: [
    BlocProvider<LanguageBloc>.value(
      value: _languageBloc,
    ),
    BlocProvider<LoginBloc>.value(
      value: _loginBloc,
    ),
  ]
Declare the bloc in the state class.
Initialise the bloc from getIt, in initState().
Dispose of the bloc in the dispose().
Use the MultiBlocProvider now to accommodate both LanguageBloc and LoginBloc.
Handle the Login Request
Now it’s time to hit the API and see how we show errors and handle success in the app.

Open LoginForm and dispatch the event for LoginBloc when the button is pressed.

onPressed: enableSignIn
  ? () {
      BlocProvider.of<LoginBloc>(context).add(
        LoginInitiateEvent(
          _userNameController.text,
          _passwordController.text,
        ),
      );
    }
  : null,
Dispatch the loginInitiateEvent with the username and password from the controllers.
Run the app by giving any random username and password. You will receive wrongUsernamePassword as expected. So, let's consume this response. There is a reason why I am saying consume because we have to deal with 2 scenarios. First scenario when there is an error thrown and we don't navigate anywhere but show the error message to the user, for this, we generally use BlocBuilder. In the second scenario when we got success from login and we navigate to the home screen, for this we can use BlocListener. We need both these functionalities in LoginForm itself. So, when we need both BlocListener and BlocBuilder, we can use 1 widget BlocConsumer which can listen as well as build.

Let’s do that above the sign in button.

//1
BlocConsumer<LoginBloc, LoginState>(
  //2
  buildWhen: (previous, current) => current is LoginError,
  //3
  builder: (context, state) {
    if (state is LoginError)
      return Text(
        state.message.t(context),
        style: Theme.of(context).textTheme.orangeSubtitle1,
      );
    return const SizedBox.shrink();
  },
  //4
  listenWhen: (previous, current) => current is LoginSuccess,
  //5
  listener: (context, state) {
    Navigator.of(context).pushNamedAndRemoveUntil(
      RouteList.home,
      (route) => false,
    );
  },
)
Add the BlocConsumer for LoginBloc.
Use buildWhen to reduce calls to builder. The condition to build is when the current state is LoginError.
In build() return a Text widget with the error message returned by the LoginError state. Now, when you run the application, you will see an error message displayed when there is an error from API call. We don't have a loader currently in the app so, wait for 1 or 2 articles where we add loaded in all the screen when an API is hit.
Use listenWhen to reduce calls to listener. We only listen when the state is a success.
In listener we will navigate to the home screen. Here, we will use pushNamedAndRemoveUntil() because if the user presses the back button from the home screen he should not navigate back to the login screen.
Logout
Now, we have done login. So, let’s add the logout item in the navigation drawer and add usecase, repository methods, data source methods in one go. And bind them with the bloc.

Open NavigationDrawer and logout as the last item.

NavigationListItem(
  title: TranslationConstants.logout.t(context),
  onPressed: () {
    //1
    BlocProvider.of<LoginBloc>(context).add(LogoutEvent());
  },
)
On the press of Logout we will dispatch the LogoutEvent.
Let’s move to the bloc. Open LoginBloc and add a LogoutUser usecase along with LoginUser. We haven't created it yet, but just after bloc, we are moving to that part.

final LogoutUser logoutUser;
LoginBloc({
  @required this.loginUser,
  @required this.logoutUser,
}) : super(LoginInitial());
//1
else if (event is LogoutEvent) {
  await logoutUser(NoParams());
  yield LogoutSuccess();
}
After declaring and putting in the constructor, you’ll handle the LogoutEvent. Here, you'll always yield LogoutSuccess after the API call.
Let’s get back to NavigationDrawer to handle this state and navigate the user to the login screen.

//1
BlocListener<LoginBloc, LoginState>(
  //2
  listenWhen: (previous, current) => current is LogoutSuccess,
  //3
  listener: (context, state) {
    Navigator.of(context).pushNamedAndRemoveUntil(
        RouteList.initial, (route) => false);
  },
  child: NavigationListItem(
    title: TranslationConstants.logout.t(context),
    onPressed: () {
      BlocProvider.of<LoginBloc>(context).add(LogoutEvent());
    },
  ),
),
Wrap the NavigationListItem with BlocListener.
Only listen when the state is LogoutSuccess.
In listener, you can again use pushNamedAndRemoveUntil to move to the initial screen and exit out of the app when the back button is pressed.
Let’s see the usecase, repository, and data source now.

Create a new usecase LogoutUser

//1
class LogoutUser extends UseCase<void, NoParams> {
  final AuthenticationRepository _authenticationRepository;
  LogoutUser(this._authenticationRepository);
  //2
  @override
  Future<Either<AppError, void>> call(NoParams noParams) async =>
      _authenticationRepository.logoutUser();
}
This usecase will return nothing and will take NoParams. Like LoginUser, this will also use AuthenticationRepository
In the call(), we will invoke the logoutUser(), which we will see next.
Let’s add code in the implementation of logoutUser().

@override
Future<Either<AppError, void>> logoutUser() async {
  //1
  final sessionId = await _authenticationLocalDataSource.getSessionId();
  //2
  await Future.wait([
    _authenticationRemoteDataSource.deleteSession(sessionId),
    _authenticationLocalDataSource.deleteSessionId(),
  ]);
  print(await _authenticationLocalDataSource.getSessionId());
  //3
  return Right(Unit);
}
We haven’t created any of the methods, yes to delete from local DB or API call to delete the session, but we will now create. Before that, let’s invoke them in the repository.

First, we will fetch the sessionId from the local database.
Make 2 await calls in 1 call using Future.wait, by this they will be called parallelly. We will parallelly delete the session on TMDb API and our local database. To be sure, we will print the session id from local to debug further.
Since, this method return always success we will return the Right object with Unit.
Let’s work on local database call first for getSessionId and deleteSessionId.

@override
Future<void> deleteSessionId() async {
  print('delete session - local');
  //1
  final authenticationBox = await Hive.openBox('authenticationBox');
  authenticationBox.delete('session_id');
}
@override
  //2
  final authenticationBox = await Hive.openBox('authenticationBox');
  return await authenticationBox.get('session_id');
}
In the deleteSessionId(), we will get the box first and delete the session_id key.
In the getSessionId(), we will again get the box and get the value stored by the session_id key.
Let’s make the delete API call now in the remote data source.

//1
Future<bool> deleteSession(String sessionId);
@override
Future<bool> deleteSession(String sessionId) async {
  //2
  final response = await _client.deleteWithBody(
    'authentication/session',
    params: {
      'session_id': sessionId,
    },
  );
  //3
  return response['success'] ?? false;
}
Create the abstract method now to delete the session id.
We haven’t created yet the delete API call with body in our ApiClient, so let's just add it here. This method will make a DELETE Api call with path and body parameters.
After the API call, based on success true or false, we will return the bool value.
Delete API Call
Delete API call is a little different from other calls like GET and POST. Here, the default HttpClient doesn't provide a way to pass body params in the call itself. So, we first create a Request object and attach the body and header to it.

Open ApiClient

//1
dynamic deleteWithBody(String path, {Map<dynamic, dynamic> params}) async {
  //2
  Request request = Request('DELETE', Uri.parse(getPath(path, null)));
  //3
  request.headers['Content-Type'] = 'application/json';
  //4
  request.body = jsonEncode(params);
  //5
  final response = await _client.send(request).then(
        (value) => Response.fromStream(value),
      );
  //6
  if (response.statusCode == 200) {
    return json.decode(response.body);
  } else if (response.statusCode == 401) {
    throw UnauthorisedException();
  } else {
    throw Exception(response.reasonPhrase);
  }
}
Create the deleteWithBody with path and params.
Create a Request object with the method passed as DELETE and path as we do for GET and POST methods.
Add the header to the request now.
Add the body as well to the request.
Send the request now and get the response from the StreamedResponse.
Now, handle the response based on statusCode, as we have done at the start of the application.
Let’s hit the logout button now and see the overall flow.

So here is a quick recap of what we did in this article. Create a POST API Call. Invoke it from datasource, repository, and usecase. Create a screen to take your username and password. Using bloc, we made the API call to login user and enter into the app after saving the session id in local. Then, we logged him out by deleting the session id from local and remote.
