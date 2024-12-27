Here’s a **comprehensive guide to Flutter Clean Architecture** that:
- Uses **Dio** for API requests.
- Uses **BLoC** for state management.
- Uses **GetIt** for dependency injection.
- Follows a **separated layer structure** adhering to Clean Architecture principles.

---

### **Folder Structure**
We will follow a modular and scalable folder structure:

```
lib/
├── core/
│   ├── network/
│   │   ├── api_constants.dart       # API URLs and endpoints
│   │   ├── dio_client.dart          # Central Dio configuration
│   │   ├── api_interceptor.dart     # Handles errors and headers
│   ├── result.dart                  # Result class for consistent responses
├── features/
│   ├── auth/                        # Authentication Feature
│   │   ├── data/
│   │   │   ├── auth_service.dart       # Handles API requests
│   │   │   ├── auth_repository_impl.dart # Implements Repository
│   │   ├── domain/
│   │   │   ├── repositories/
│   │   │   │   ├── auth_repository.dart # Repository interface
│   │   │   ├── use_cases/
│   │   │   │   ├── login_use_case.dart # Login Use Case
│   │   ├── presentation/
│   │   │   ├── bloc/
│   │   │   │   ├── login_bloc.dart     # Handles login logic
│   │   │   ├── pages/
│   │   │   │   ├── login_page.dart    # Login UI
│   ├── home/                        # Home Feature
│   │   ├── presentation/
│   │   │   ├── pages/
│   │   │   │   ├── home_page.dart    # Home UI
├── dependency_injection.dart         # Dependency injection setup
├── main.dart                         # App entry point
```

---

### **Step 1: Core Layer**

#### **a. Result Class**
A utility class to wrap API responses.

##### `core/result.dart`
```dart
class Result<T> {
  final T? data;
  final String? message;
  final bool isSuccess;

  Result({this.data, this.message, required this.isSuccess});

  factory Result.success(T data) => Result(data: data, isSuccess: true);

  factory Result.error(String message) => Result(message: message, isSuccess: false);
}
```

---

#### **b. Dio Configuration**
Centralize Dio setup with interceptors.

##### `core/network/api_constants.dart`
```dart
class ApiConstants {
  static const String baseUrl = 'https://api.example.com/';
  static const String loginEndpoint = 'auth/login';
}
```

##### `core/network/api_interceptor.dart`
```dart
import 'package:dio/dio.dart';

class ApiInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final token = 'your-auth-token'; // Replace with token logic
    if (token.isNotEmpty) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    super.onRequest(options, handler);
  }

  @override
  void onError(DioError error, ErrorInterceptorHandler handler) {
    // Handle custom error messages
    final errorMessage = _handleError(error);
    handler.next(DioError(
      requestOptions: error.requestOptions,
      response: error.response,
      type: error.type,
      error: errorMessage,
    ));
  }

  String _handleError(DioError error) {
    if (error.type == DioErrorType.connectionTimeout) {
      return 'Connection timeout. Please try again.';
    }
    if (error.response?.statusCode == 401) {
      return 'Unauthorized. Please login again.';
    }
    return 'Unexpected error occurred.';
  }
}
```

##### `core/network/dio_client.dart`
```dart
import 'package:dio/dio.dart';
import 'api_interceptor.dart';
import 'api_constants.dart';

class DioClient {
  late final Dio _dio;

  DioClient() {
    _dio = Dio(
      BaseOptions(
        baseUrl: ApiConstants.baseUrl,
        connectTimeout: const Duration(seconds: 30),
        receiveTimeout: const Duration(seconds: 30),
        headers: {
          'Accept': 'application/json',
        },
      ),
    );

    _dio.interceptors.add(ApiInterceptor());
  }

  Dio get dio => _dio;
}
```

---

### **Step 2: Domain Layer**

#### **a. Repository Interface**
Defines the contract for the repository.

##### `features/auth/domain/repositories/auth_repository.dart`
```dart
import '../../../../core/result.dart';

abstract class AuthRepository {
  Future<Result<bool>> login(String email, String password);
}
```

#### **b. Use Case**
Encapsulates business logic.

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

### **Step 3: Data Layer**

#### **a. AuthService**
Handles API requests.

##### `features/auth/data/auth_service.dart`
```dart
import 'package:dio/dio.dart';
import '../../../core/network/api_constants.dart';
import '../../../core/result.dart';

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
Implements the repository interface.

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
      return Result.success(isSuccess);
    } catch (e) {
      return Result.error('Login failed: $e');
    }
  }
}
```

---

### **Step 4: Presentation Layer**

#### **a. Login BLoC**
Handles login logic.

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
        emit(LoginState(error: result.message ?? 'Unknown error occurred.'));
      }
    });
  }
}
```

---

#### **b. Login Page**
Builds the login UI.

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
                  ),
                  TextField(
                    decoration: const InputDecoration(labelText: 'Password'),
                    obscureText: true,
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

### **Step 5: Dependency Injection**

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

### **Step 6: App Entry Point**

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
      title: 'Flutter Clean Architecture',
      theme: ThemeData(primarySwatch: Colors.blue),
      initialRoute: '/',
      routes: {
        '/': (context) => const LoginPage(),
        '/home': (context) => const Scaffold(body: Center(child: Text('Home Page'))),
      },
    );
  }
}
```

---

### **Summary**

This guide demonstrates:
1. **Dio** for network requests with interceptors.
2. **BLoC** for managing state and business logic.
3. **GetIt** for dependency injection.
4. A fully modular **Clean Architecture** setup with separated layers.

You can now build on this foundation by adding more features, such as user registration, token refresh, or additional pages! Let me know if you have further questions. 😊
