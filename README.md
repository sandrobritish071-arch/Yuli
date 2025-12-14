import 'package:flutter/material.dart';
import 'package:local_auth/local_auth.dart';
import 'package:flutter/services.dart';

void main() {
  // Убедитесь, что Flutter инициализирован перед использованием LocalAuthentication
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const BiometricApp());
}

class BiometricApp extends StatelessWidget {
  const BiometricApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Enhanced Biometric Auth',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        primarySwatch: Colors.indigo,
        useMaterial3: true,
      ),
      home: const FingerprintAuthScreen(),
    );
  }
}

class FingerprintAuthScreen extends StatefulWidget {
  const FingerprintAuthScreen({super.key});

  @override
  State<FingerprintAuthScreen> createState() => _FingerprintAuthScreenState();
}

class _FingerprintAuthScreenState extends State<FingerprintAuthScreen> {
  final LocalAuthentication auth = LocalAuthentication();
  String _authStatus = 'Нажмите для проверки доступа';
  bool _isAuthenticating = false; // Флаг для предотвращения повторных нажатий
  bool _canAuthenticate = false; // Флаг доступности биометрии

  @override
  void initState() {
    super.initState();
    _checkBiometricsSupport();
  }
  
  // --- 1. ПРОВЕРКА ДОСТУПНОСТИ ---

  Future<void> _checkBiometricsSupport() async {
    bool canCheckBiometrics = false;
    List<BiometricType> availableBiometrics = [];
    
    try {
      canCheckBiometrics = await auth.canCheckBiometrics;
      if (canCheckBiometrics) {
        availableBiometrics = await auth.getAvailableBiometrics();
      }
    } on PlatformException catch (e) {
      print('Ошибка при проверке биометрии: $e');
    }

    if (!mounted) return;

    setState(() {
      _canAuthenticate = canCheckBiometrics && availableBiometrics.isNotEmpty;
      if (_canAuthenticate) {
        // Указываем, какие методы доступны (например, Face ID, Fingerprint)
        _authStatus = 'Готов к аутентификации. Доступно: ${availableBiometrics.map((t) => t.toString().split('.').last).join(', ')}';
      } else {
        _authStatus = '❌ Биометрия недоступна или не настроена на устройстве.';
      }
    });
  }

  // --- 2. ЗАПУСК АУТЕНТИФИКАЦИИ С РЕЗЕРВОМ ---

  Future<void> _authenticate() async {
    if (!_canAuthenticate || _isAuthenticating) return;
    
    setState(() {
      _isAuthenticating = true;
      _authStatus = 'Попытка аутентификации...';
    });

    bool authenticated = false;
    try {
      authenticated = await auth.authenticate(
        localizedReason: 'Подтвердите свою личность для входа в приложение',
        options: const AuthenticationOptions(
          stickyAuth: true,
          useErrorDialogs: true,
          // *** Улучшение: Разрешаем резервный метод (PIN-код/Пароль) ***
          biometricOnly: false, 
        ),
      );
    } on PlatformException catch (e) {
      // Обработка конкретных ошибок
      if (e.code == 'NotEnrolled') {
        _authStatus = '❌ Ошибка: Биометрия не настроена.';
      } else if (e.code == 'LockedOut') {
        _authStatus = '❌ Ошибка: Слишком много попыток. Заблокировано.';
      } else {
        _authStatus = '❌ Ошибка платформы: ${e.message}';
      }
      authenticated = false;
    }

    if (!mounted) return;

    setState(() {
      _isAuthenticating = false;
      if (authenticated) {
        _authStatus = '✅ Успешно. Доступ разрешен.';
        // Здесь обычно происходит переход на главный экран приложения:
        // Navigator.pushReplacement(context, MaterialPageRoute(builder: (context) => HomeScreen()));
      } else if (!_authStatus.contains('Ошибка')) {
        // Обновляем статус, если не была поймана специфическая ошибка
        _authStatus = '❌ Аутентификация не удалась или отменена.';
      }
    });
  }

  // --- UI ---

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Биометрический вход'),
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
      ),
      body: Center(
        child: Padding(
          padding: const EdgeInsets.all(24.0),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: <Widget>[
              // Динамическая иконка
              Icon(
                _isAuthenticating ? Icons.lock_open : Icons.lock_outline,
                size: 80,
                color: _canAuthenticate ? Colors.indigo : Colors.grey,
              ),
              const SizedBox(height: 30),
              // Статус
              Text(
                _authStatus,
                textAlign: TextAlign.center,
                style: TextStyle(fontSize: 18, color: _authStatus.startsWith('✅') ? Colors.green : Colors.white70),
              ),
              const SizedBox(height: 80),
              // Кнопка аутентификации
              ElevatedButton.icon(
                onPressed: _authenticate,
                icon: const Icon(Icons.fingerprint),
                label: Text(
                  _isAuthenticating ? 'Проверка...' : 'Войти по биометрии',
                  style: const TextStyle(fontSize: 18),
                ),
                style: ElevatedButton.styleFrom(
                  backgroundColor: _canAuthenticate ? Colors.indigo : Colors.grey,
                  foregroundColor: Colors.white,
                  padding: const EdgeInsets.symmetric(horizontal: 40, vertical: 15),
                  elevation: 5,
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
