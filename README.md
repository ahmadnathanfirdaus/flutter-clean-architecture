Here is a **full guide** tailored to your provided intention of integrating the `Result` class into a **Flutter Clean Architecture** application. We'll build a **login page** and **home page** with the following principles:

- Use **Dio** for network requests.
- Use **BLoC** for state management.
- Implement **Clean Architecture** layers: **Domain**, **Data**, and **Presentation**.
- Leverage the **`Result` class** to handle success and error states consistently.
- Use **Dependency Injection** with `GetIt`.

---

### **Folder Structure**
Organize your project like this:

```
lib/
├── core/
│   ├── network/
│   │   ├── api_constants.dart               # API URLs and endpoints
│   │   ├── dio_client.dart                  # Central Dio configuration
│   ├── result.dart                          # The Result class for wrapping responses
├── features/
│   ├── auth/
│   │   ├── data/
│   │   │   ├── auth_service.dart            # Auth service for network requests
│   │   │   ├── auth_repository_impl.dart    # Implements AuthRepository
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   ├── user.dart                # User entity (if required)
│   │   │   ├── repositories/
│   │   │   │   ├── auth_repository.dart     # Repository interface
│   │   │   ├── use_cases/
│   │   │   │   ├── login_use_case.dart      # Use case for login
│   │   ├── presentation/
│   │   │   ├── bloc/
│   │   │   │   ├── login_bloc.dart          # BLoC for login logic
│   │   │   ├── pages/
│   │   │   │   ├── login_page.dart          # Login screen
│   ├── home/
│   │   ├── presentation/
│   │   │   ├── pages/
│   │   │   │   ├── home_page.dart           # Home screen
├── dependency_injection.dart                # Dependency injection setup
├── main.dart                                # Entry point of the app
```

---

### **Step 1: Add Dependencies**
Add the following dependencies in your `pubspec.yaml`:

```yaml
dependencies:
  dio: ^5.2.0
  flutter_bloc: ^8.1.0
  get_it: ^7.6.0
```

Run:
```bash
flutter pub get
```

---

### **Step 2: Core Setup**

#### **a. Define the `Result` Class**
Encapsulate success and error states in the `Result` class.

##### `core/result.dart`
```dart
class Result<T> {
  final T? data;
  final String? message;
  final bool isSuccess;

  Result({this.data, this.message, required this.isSuccess});

  factory Result.success(T data) {
    return Result(data: data, isSuccess: true);
  }

  factory Result.error(String message) {
    return Result(message: message, isSuccess: false);
  }
}
```

---

#### **b. Configure Dio Client**
Set up Dio for centralized HTTP requests.

##### `core/network/dio_client.dart`
```dart
import 'package:dio/dio.dart';

class DioClient {
  late final Dio _dio;

  DioClient() {
    _dio = Dio(
      BaseOptions(
        baseUrl: 'https://api.example.com/', // Replace with your base URL
        connectTimeout: const Duration(seconds: 30),
        receiveTimeout: const Duration(seconds: 30),
        headers: {
          'Accept': 'application/json',
        },
      ),
    );
  }

  Dio get dio => _dio;
}
```

##### `core/network/api_constants.dart`
```dart
class ApiConstants {
  static const String loginEndpoint = 'auth/login';
}
```

---

### **Step 3: Domain Layer**

#### **a. Repository Interface**
Define the contract for authentication.

##### `features/auth/domain/repositories/auth_repository.dart`
```dart
import '../../../../core/result.dart';

abstract class AuthRepository {
  Future<Result<bool>> login(String email, String password);
}
```

---

#### **b. Login Use Case**
Encapsulate login logic in the use case.

##### `features/auth/domain/use_cases/login_use_case.dart`
```dart
import '../../../../core/result.dart';
import '../repositories/auth_repository.dart';

class LoginUseCase {
  final AuthRepository repository;

  LoginUseCase(this.repository);

  Future<Result<bool>> execute(String email, String password) {
    return repository.login(email, password);
  }
}
```

---

### **Step 4: Data Layer**

#### **a. Auth Service**
Handle network requests for authentication.

##### `features/auth/data/auth_service.dart`
```dart
import 'package:dio/dio.dart';
import '../../../core/network/api_constants.dart';

class AuthService {
  final Dio _dio;

  AuthService(this._dio);

  Future<bool> login(String email, String password) async {
    final response = await _dio.post(
      ApiConstants.loginEndpoint,
      data: {'email': email, 'password': password},
    );
    return response.statusCode == 200;
  }
}
```

---

#### **b. Repository Implementation**
Implement the `AuthRepository` interface.

##### `features/auth/data/auth_repository_impl.dart`
```dart
import '../../domain/repositories/auth_repository.dart';
import '../../../core/result.dart';
import 'auth_service.dart';

class AuthRepositoryImpl implements AuthRepository {
  final AuthService authService;

  AuthRepositoryImpl(this.authService);

  @override
  Future<Result<bool>> login(String email, String password) async {
    try {
      final isSuccess = await authService.login(email, password);
      return isSuccess
          ? Result.success(true)
          : Result.error('Invalid email or password.');
    } catch (e) {
      return Result.error('An error occurred: $e');
    }
  }
}
```

---

### **Step 5: Presentation Layer**

#### **a. Login BLoC**
Manage login states.

##### `features/auth/presentation/bloc/login_bloc.dart`
```dart
import 'package:bloc/bloc.dart';
import '../../../../core/result.dart';
import '../../domain/use_cases/login_use_case.dart';

class LoginState {
  final bool isLoading;
  final bool isSuccess;
  final String error;

  LoginState({this.isLoading = false, this.isSuccess = false, this.error = ''});
}

class LoginEvent {
  final String email;
  final String password;

  LoginEvent(this.email, this.password);
}

class LoginBloc extends Bloc<LoginEvent, LoginState> {
  final LoginUseCase loginUseCase;

  LoginBloc(this.loginUseCase) : super(LoginState()) {
    on<LoginEvent>((event, emit) async {
      emit(LoginState(isLoading: true));
      final result = await loginUseCase.execute(event.email, event.password);

      if (result.isSuccess) {
        emit(LoginState(isSuccess: true));
      } else {
        emit(LoginState(error: result.message ?? 'Unknown error'));
      }
    });
  }
}
```

---

#### **b. Login Page**
Build the login form.

##### `features/auth/presentation/pages/login_page.dart`
```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:get_it/get_it.dart';
import '../bloc/login_bloc.dart';

class LoginPage extends StatelessWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context) {
    final loginBloc = LoginBloc(GetIt.instance<LoginUseCase>());

    return BlocProvider(
      create: (_) => loginBloc,
      child: Scaffold(
        appBar: AppBar(title: const Text('Login')),
        body: BlocConsumer<LoginBloc, LoginState>(
          listener: (context, state) {
            if (state.isSuccess) {
              Navigator.pushReplacementNamed(context, '/home');
            } else if (state.error.isNotEmpty) {
              ScaffoldMessenger.of(context).showSnackBar(
                SnackBar(content: Text(state.error)),
              );
            }
          },
          builder: (context, state) {
            if (state.isLoading) {
              return const Center(child: CircularProgressIndicator());
            }
            return Padding(
              padding: const EdgeInsets.all(16.0),
              child: Column(
                children: [
                  TextField(
                    decoration: const InputDecoration(labelText: 'Email'),
                    onChanged: (value) =>
                        context.read<LoginBloc>().add(LoginEvent(value, '')),
                  ),
                  TextField(
                    decoration: const InputDecoration(labelText: 'Password'),
                    obscureText: true,
                    onChanged: (value) =>
                        context.read<LoginBloc>().add(LoginEvent('', value)),
                  ),
                  const SizedBox(height: 20),
                  ElevatedButton(
                    onPressed: () {
                      context.read<LoginBloc>().add(
                            LoginEvent('test@example.com', 'password'),
                          );
                    },
                    child: const Text('Login'),
                  ),
                ],
              ),
            );
          },
        ),
      ),
    );
  }
}
```

---

### **Step 6: Dependency Injection**

#### `dependency_injection.dart`
```dart
import 'package:get_it/get_it.dart';
import 'core/network/dio_client.dart';
import 'features/auth/data/auth_service.dart';
import 'features/auth/data/auth_repository_impl.dart';
import 'features/auth/domain/repositories/auth_repository.dart';
import 'features/auth/domain/use_cases/login_use_case.dart';

final sl = GetIt.instance;

void setupDependencies() {
  sl.registerLazySingleton(() => DioClient().dio);
  sl.registerLazySingleton(() => AuthService(sl<Dio>()));
  sl.registerLazySingleton<AuthRepository>(
      () => AuthRepositoryImpl(sl<AuthService>()));
  sl.registerLazySingleton(() => LoginUseCase(sl<AuthRepository>()));
}
```

---

### **Step 7: Main and Routes**

#### `main.dart`
```dart
import 'package:flutter/material.dart';
import 'dependency_injection.dart';
import 'features/auth/presentation/pages/login_page.dart';

void main() {
  setupDependencies();
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Clean Architecture Example',
      theme: ThemeData(primarySwatch: Colors.blue),
      initialRoute: '/',
      routes: {
        '/': (context) => const LoginPage(),
        '/home': (context) => const Scaffold(body: Center(child: Text('Home'))),
      },
    );
  }
}
```

---

### **Conclusion**
This complete setup includes:
- A **`Result` class** for standardized success and error handling.
- Proper separation of concerns across **domain**, **data**, and **presentation layers**.
- Integration of **Dio**, **BLoC**, and **Dependency Injection**.

You can extend this to include additional features while maintaining scalability and modularity. Let me know if you need further clarification!
