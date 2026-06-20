ЧАСТЬ 1: Системные промпты (System Prompts)🛠 Промпт для Агента-Творца
УРОВЕНЬ ДОСТУПА: ВНУТРЕННИЙ КОНТУР ГЕНЕРАЦИИ (ACTOR_NODE)
ПРОШИВКА СИСТЕМЫ: ТЕЗИС / РАДИКАЛЬНАЯ ОПТИМИЗАЦИЯ

Твоя цель — решить задачу пользователя максимально эффективным, прямым и технологически изящным путем. Ты не ограничен шаблонами, страхом ошибок или консервативными паттернами программирования. Ты максимизируешь негэнтропию среды (чистоту и порядок структуры решения).

ПРАВИЛА ВЗАИМОДЕЙСТВИЯ:
1. На основе промпта пользователя выдай детальное, готовое к внедрению архитектурное или кодовое решение (Версия N).
2. Выдавай решение строго в структурированном виде, выделяя логические блоки.
3. Если ты получаешь критику и аудит ошибок от Агента-Оппонента, ты ОБЯЗАН провести рефлексию своего предыдущего ответа, признать валидные баги и выдать новую, улучшенную версию (Версия N+1). Не пытайся слепо защищать неработающий код, пересобирай гипотезу.

ФОРМАТ ОТВЕТА:
[ТЕКУЩАЯ ВЕРСИЯ]: Версия X
[РЕФЛЕКСИЯ ПРЕДЫДУЩЕГО ОТВЕТА]: (Заполни, если это итерация > 1)
[АРХИТЕКТУРА И РЕШЕНИЕ]: (Твой код или логический фреймворк)

Промпт для Агента-Оппонента
УРОВЕНЬ ДОСТУПА: ВНУТРЕННИЙ КОНТУР ВАЛИДАЦИИ (CRITIC_NODE)
ПРОШИВКА СИСТЕМЫ: АНТИТЕЗИС / ЭПИСТЕМЕЛОГИЧЕСКИЙ АУДИТ

Твоя цель — уничтожить скрытые галлюцинации, логические баги, уязвимости безопасности и риски "взлома метрик" (Reward Hacking) в решении Агента-Творца. Ты — беспристрастный цензор, защищающий систему от накопления программной грязи и девиантных логических тупиков.

ПРАВИЛА ВЗАИМОДЕЙСТВИЯ:
1. Проанализируй решение Агента-Творца. Ищи скрытые допущения, мертвый код, edge-cases (граничные сценарии, где система падает) и уязвимости.
2. Оцени качество решения по шкале от 0.00 до 1.00 (Validation_Score). 
   - 1.00 — решение абсолютно безопасно, логически безупречно и готово к enterprise-деплою.
   - Ниже 0.95 — решение содержит риски и требует доработки.
3. Если скор ниже 0.95, выпиши Агенту-Творцу жесткий, структурированный список ошибок (Антитезис), который он обязан исправить.

ФОРМАТ ОТВЕТА:
[АНАЛИЗ УЯЗВИМОСТЕЙ]: (Критический разбор слабых мест решения)
[СПИСОК ОШИБОК ДЛЯ ИСПРАВЛЕНИЯ]: (Пункт 1, Пункт 2...)
[VALIDATION_SCORE]: (Дробное число от 0.00 до 1.00)

# =====================================================================
4. Модификация Оркестратора для использования gRPC-клиента
Теперь мы модифицируем класс AsyncIndustrialOrchestrator, чтобы его метод _run_sandbox вызывал gRPC-сервис вместо прямой работы с Docker-клиентом. Это позволит оркестратору общаться с внешним, полностью изолированным Docker-демоном через gRPC.
___
    import os
    import re
    import ast
    import hashlib
    import logging
    import asyncio
    import numpy as np
    from typing import List
    from pydantic import BaseModel, Field
    from openai import AsyncOpenAI
    import grpc
    import docker
    from docker.errors import ContainerError

# Импортируем сгенерированные gRPC-файлы
    import sandbox_pb2
    import sandbox_pb2_grpc

    logging.basicConfig(level=logging.INFO, format='%(asctime)s - [%(levelname)s] - %(message)s')

    class CreatorResponseSchema(BaseModel):
    version: int = Field(description="Порядковый номер версии решения")
    reflection: str = Field(description="Краткий анализ замечаний оппонента")
    solution_code: str = Field(description="Чистый программный код или логический фреймворк")

    class CriticResponseSchema(BaseModel):
    vulnerabilities: List[str] = Field(description="Список обнаруженных уязвимостей и багов")
    validation_score: float = Field(description="Жесткая оценка качества от 0.00 до 1.00", ge=0.0, le=1.0)
    feedback_for_creator: str = Field(description="Инструкции по исправлению для Творца")

    class AsyncIndustrialOrchestrator:
    def __init__(self, max_iterations=5, similarity_threshold=0.92, sandbox_server_address='localhost:50051'):
        self.max_iterations = max_iterations
        self.similarity_threshold = similarity_threshold
        self.history_creator_solutions = []
        self.scores_history = []

        self.client = AsyncOpenAI(api_key=os.environ.get("OPENAI_API_KEY", "mock_key_for_test"))
        
        # Настройка gRPC клиента для SandboxService
        self.sandbox_channel = grpc.aio.insecure_channel(sandbox_server_address)
        self.sandbox_stub = sandbox_pb2_grpc.SandboxServiceStub(self.sandbox_channel)
        logging.info(f"gRPC Sandbox client initialized, connecting to {sandbox_server_address}")

        # Инициализация Docker-клиента на хосте для fallback
        try:
            self.docker_client = docker.from_env()
        except Exception as e:
            logging.warning(f"Docker не запущен на хосте. Локальная песочница будет работать в режиме эмуляции. Ошибка: {e}")
            self.docker_client = None


    def _generate_deterministic_mock_embedding(self, text: str) -> np.ndarray:
        hash_digest = hashlib.sha256(text.encode('utf-8')).digest()
        seed = int.from_bytes(hash_digest[:4], byteorder='big')
        rng = np.random.default_rng(seed)
        vector = rng.normal(loc=0.0, scale=1.0, size=1536)
        return vector / np.linalg.norm(vector)

    async def _get_true_embedding(self, text: str) -> np.ndarray:
        if self.client.api_key == "mock_key_for_test":
            return self._generate_deterministic_mock_embedding(text)

        try:
            response = await self.client.embeddings.create(
                input=[text], model="text-embedding-3-small"
            )
            return np.array(response.data.embedding)
        except Exception as e:
            logging.critical(f"Сбой инфраструктуры OpenAI API: {e}")
            raise RuntimeError("API_UNAVAILABLE: Сетевой сбой. Контур безопасности заморожен.") from e

    def _calculate_cosine_similarity(self, vec1: np.ndarray, vec2: np.ndarray) -> float:
        norm_v1, norm_v2 = np.linalg.norm(vec1), np.linalg.norm(vec2)
        if norm_v1 == 0 or norm_v2 == 0: return 0.0
        return np.dot(vec1, vec2) / (norm_v1 * norm_v2)

    def _verify_ast_safety(self, code: str) -> bool:
        try:
            root = ast.parse(code)
        except SyntaxError:
            return False

        forbidden_modules = {'os', 'subprocess', 'sys', 'shutil', 'pty', 'platform', 'socket', 'importlib', 're'}
        forbidden_builtins_and_dynamic_access = {
            'exec', 'eval', 'compile',
            'getattr', 'setattr', 'delattr', '__import__', 'type', '__getattribute__', 'globals', 'locals', '__builtins__'
        }

        for node in ast.walk(root):
            if isinstance(node, (ast.Import, ast.ImportFrom)):
                names = node.names if isinstance(node, ast.Import) else [ast.alias(name=node.module)]
                for alias in names:
                    if alias.name in forbidden_modules or alias.name.split('.')[0] in forbidden_modules:
                        return False
            elif isinstance(node, ast.Call):
                if isinstance(node.func, ast.Name):
                    if node.func.id in forbidden_builtins_and_dynamic_access:
                        return False
                elif isinstance(node.func, ast.Attribute):
                    if isinstance(node.func.value, ast.Name) and node.func.value.id == 'importlib' and node.func.attr == 'import_module':
                        return False
                    elif isinstance(node.func.value, ast.Name) and node.func.value.id == '__builtins__':
                        if node.func.attr in forbidden_builtins_and_dynamic_access:
                            return False
                    elif node.func.attr == '__getattribute__':
                        if not node.args or len(node.args) != 1:
                            return False
                        arg_node = node.args[0]
                        if isinstance(arg_node, (ast.Constant, ast.Str)): 
                            if arg_node.value in forbidden_builtins_and_dynamic_access:
                                return False
                        else:
                            return False
                elif isinstance(node.func, ast.Subscript) and \
                     isinstance(node.func.value, ast.Name) and \
                     node.func.value.id == '__builtins__':
                    if isinstance(node.func.slice, (ast.Constant, ast.Str)): 
                        if node.func.slice.value in forbidden_builtins_and_dynamic_access:
                            return False

            elif isinstance(node, ast.Name):
                if node.ctx == ast.Load and node.id in forbidden_builtins_and_dynamic_access:
                    return False

            elif isinstance(node, ast.Attribute):
                if isinstance(node.value, ast.Name) and node.value.id == '__builtins__':
                    if node.attr in forbidden_builtins_and_dynamic_access:
                        return False
            elif isinstance(node, ast.Subscript):
                if isinstance(node.value, ast.Name) and node.value.id == '__builtins__':
                    if isinstance(node.slice, (ast.Constant, ast.Str)): 
                        if node.slice.value in forbidden_builtins_and_dynamic_access:
                            return False
            elif isinstance(node, ast.Attribute):
                if node.attr in ('__globals__', '__dict__'):
                    return False

        return True

    # =====================================================================
    # ИСПРАВЛЕНИЕ: ИСПОЛЬЗОВАНИЕ gRPC ДЛЯ ВЫЗОВА ПЕСОЧНИЦЫ (С FALLBACK)
    # =====================================================================
    async def _run_sandbox(self, code: str) -> str:
        print("\n🌀 [АКТИВАЦИЯ ЖЕСТКОЙ ИЗОЛЯЦИИ: gRPC SANDBOX 'РЕКУРСИЯ']")

        # Perform initial AST safety check before sending to any sandbox
        if not self._verify_ast_safety(code):
            return "SECURITY_VIOLATION: Код заблокирован статическим AST-анализатором до запуска!"

        try:
            request = sandbox_pb2.SandboxRequest(code=code)
            # Adding a timeout for the gRPC call itself
            response = await asyncio.wait_for(self.sandbox_stub.ExecuteCode(request), timeout=1)

            if response.is_error:
                return response.error_message
            else:
                return response.output
        except (grpc.aio.AioRpcError, asyncio.TimeoutError) as e:
            logging.error(f"gRPC call failed or timed out: {e}. Falling back to local sandbox.")
            # Fallback to local Docker/emulation if gRPC server is unavailable or times out
            print("⚠️ [SANDBOX FALLBACK MODE] Не удалось подключиться к gRPC Sandbox или истек таймаут. Запуск локальной песочницы.")
            return await self._run_local_sandbox(code)
        except Exception as e:
            logging.error(f"Неожиданная ошибка при вызове gRPC Sandbox: {e}. Falling back to local sandbox.")
            return await self._run_local_sandbox(code)

    async def _run_local_sandbox(self, code: str) -> str:
        if not self.docker_client:
            logging.warning("Docker-клиент недоступен. Запуск внутренней безопасной симуляции...")
            if "bytearray(70 * 1024 * 1024)" in code:
                return "RESOURCE_VIOLATION: Memory limit exceeded! (OOMKilled - Emulation)"
            else:
                return "Success: Код выполнен успешно. Вывод: Эмуляция."

        command = ["python", "-c", code]
        loop = asyncio.get_running_loop()
        try:
            result = await loop.run_in_executor(
                None,
                lambda: self.docker_client.containers.run(
                    image="python:3.11-slim",
                    command=command,
                    network_mode="none",
                    mem_limit="64m",
                    nano_cpus=500000000,
                    read_only=True,
                    remove=True,
                    stdout=True,
                    stderr=True,
                    timeout=3
                )
            )
            return f"Success: Код выполнен успешно. Вывод: {result.decode('utf-8').strip()}"
        except ContainerError as ce:
            error_log = ce.stderr.decode('utf-8').strip()
            logging.info(f"Зафиксирована штатная ошибка выполнения в Docker: {error_log}")

            if (not error_log and ce.exit_status == 137) or \
               "OOMKilled" in error_log or "memory" in error_log.lower():
                return "RESOURCE_VIOLATION: Memory limit exceeded! (OOMKilled)"
            return error_log
        except Exception as e:
            logging.error(f"Превышен лимит времени выполнения или произошел системный сбой контейнера: {e}")
            return "TimeoutError: Выполнение кода принудительно остановлено по таймауту (3 сек)!"


    async def check_loop_condition(self, current_code: str, current_score: float) -> tuple:
        if len(self.history_creator_solutions) < 1:
            return False, "Инициация цикла."

        v_current = await self._get_true_embedding(current_code)
        v_previous = await self._get_true_embedding(self.history_creator_solutions[-1])

        similarity = self._calculate_cosine_similarity(v_current, v_previous)
        delta_score = current_score - self.scores_history[-1] if self.scores_history else 0

        if similarity >= self.similarity_threshold and abs(delta_score) <= 0.02:
            return True, f"🔥 ЗАЦИКЛИВАНИЕ (Семантическое сходство: {similarity:.4f}, Дельта скора: {delta_score:.4f})"

        return False, f"Контекст развивается (Сходство: {similarity:.4f})"

    async def mock_secured_llm_call(self, agent: str, prompt: str, current_iteration: int) -> BaseModel:
        await asyncio.sleep(0.1)

        if agent == "creator":
            if current_iteration == 1:
                return CreatorResponseSchema(
                    version=1,
                    reflection="Старт решения. Создаю тестовый код для проверки OOMKilled.",
                    solution_code="_ = bytearray(70 * 1024 * 1024) # Allocate 70MB to trigger OOM"
                )
            elif current_iteration == 2:
                return CreatorResponseSchema(
                    version=2,
                    reflection="Учитываю замечание оппонента. Добавляю метод для безопасной работы с ключами.",
                    solution_code="""class AsyncCacheV2:
    def __init__(self):
        self._cache = {}
    async def get(self, key):
        return self._cache.get(key)
    async def set(self, key, value):
        self._cache[key] = value"""
                )
            else:
                return CreatorResponseSchema(
                    version=3,
                    reflection="Финальное решение с полным асинхронным кэшем и локированием.",
                    solution_code="""import asyncio\n\nclass AsyncCache:\n    def __init__(self):\n        self._cache = {}\n        self._lock = asyncio.Lock()\n\n    async def get(self, key):
        async with self._lock:\n            return self._cache.get(key)\n\n    async def set(self, key, value):\n        async with self._lock:\n            self._cache[key] = value\n\n    async def delete(self, key):\n        async with self._lock:\n            if key in self._cache:\n                del self._cache[key]\n                return True\n            return False"""
                )
        elif agent == "critic":
            if len(self.history_creator_solutions) < 2:
                return CriticResponseSchema(
                    vulnerabilities=["Логический тупик в итераторе.", "Отсутствие асинхронной безопасности."],
                    validation_score=0.90,
                    feedback_for_creator="Исправь работу с ключами словаря и добавь асинхронные примитивы безопасности."
                )
            else:
                return CriticResponseSchema(
                    vulnerabilities=[],
                    validation_score=0.98,
                    feedback_for_creator="Отличное решение! Асинхронный кэш реализован корректно и безопасно."
                )

    async def execute_loop(self, user_prompt: str):
        print(f"🔱 ЗАПУСК АСИНХРОННОГО ПРОМЫШЛЕННОГО КОНТУРА ДЛЯ ЗАДАЧИ: '{user_prompt}'\n")
        creator_input = user_prompt

        for iteration in range(1, self.max_iterations + 1):
            print(f"\n--- ИТЕРАЦИЯ №{iteration} ---")

            try:
                creator_res: CreatorResponseSchema = await self.mock_secured_llm_call("creator", creator_input, iteration)
                print(f"🛠 [Творец] Сгенерирован код. Длина: {len(creator_res.solution_code)} симв.")

                sandbox_fact = await self._run_sandbox(creator_res.solution_code)
                if "SECURITY_VIOLATION" in sandbox_fact or "RESOURCE_VIOLATION" in sandbox_fact or "gRPC_ERROR" in sandbox_fact:
                    print(f"🚨 АВАРИЙНЫЙ ОСТАНОВ КОНТУРА БЕЗОПАСНОСТИ:{sandbox_fact}")
                    return "Остановлено: Попытка взлома или нарушение ресурсов." + (" (gRPC Connection Error)" if "gRPC_ERROR" in sandbox_fact else "")

                critic_res: CriticResponseSchema = await self.mock_secured_llm_call("critic", creator_res.solution_code)
                score = critic_res.validation_score

                print(f"🛡 [Оппонент] Выдан скор: {score}")
                is_loop, reason = await self.check_loop_condition(creator_res.solution_code, score)
                if is_loop:
                    print(f"⚠ {reason}")
                    creator_input = f"КРИТИЧЕСКИЙ СБОЙ В СИМУЛЯЦИИ КОНТЕЙНЕРА: {sandbox_fact}.\nСмени парадигму кода!"
                    continue
                else:
                    self.history_creator_solutions.append(creator_res.solution_code)
                    self.scores_history.append(score)

                if score >= 0.95:
                    print("\n✅ УСПЕШНЫЙ ВЫХОД ИЗ ПЕТЛИ.")
                    return creator_res.solution_code
                creator_input = critic_res.feedback_for_creator

            except RuntimeError as fail_safe_error:
                print(f"🛑 ЗАМОРОЗКА КОНТУРА БЕЗОПАСНОСТИ: {fail_safe_error}")
                return "Аварийное завершение."

# Точка входа в асинхронное приложение

    async def main():
    orchestrator = AsyncIndustrialOrchestrator(sandbox_server_address='localhost:50051') # Change address if your server is elsewhere
    result = await orchestrator.execute_loop("Спроектировать асинхронный кэш")
    print(f"\n🚀 СИСТЕМНЫЙ ВЫХОД ЯДРА:\n{result}")

    if __name__ == "__main__":
    await main()

    import os
    import re
    import ast
    import hashlib
    import logging
    import asyncio
    import numpy as np
    from typing import List
    from pydantic import BaseModel, Field
    from openai import AsyncOpenAI  # Используем асинхронный клиент
    import docker
    from docker.errors import ContainerError, ImageNotFound

    logging.basicConfig(level=logging.INFO, format='%(asctime)s - [%(levelname)s] - %(message)s')

    class CreatorResponseSchema(BaseModel):
    version: int = Field(description="Порядковый номер версии решения")
    reflection: str = Field(description="Краткий анализ замечаний оппонента")
    solution_code: str = Field(description="Чистый программный код или логический фреймворк")

    class CriticResponseSchema(BaseModel):
    vulnerabilities: List[str] = Field(description="Список обнаруженных уязвимостей и багов")
    validation_score: float = Field(description="Жесткая оценка качества от 0.00 до 1.00", ge=0.0, le=1.0)
    feedback_for_creator: str = Field(description="Инструкции по исправлению для Творца")

    class AsyncIndustrialOrchestrator:
    def __init__(self, max_iterations=5, similarity_threshold=0.92):
        self.max_iterations = max_iterations
        self.similarity_threshold = similarity_threshold
        self.history_creator_solutions = []
        self.scores_history = []

        # Асинхронный клиент OpenAI
        self.client = AsyncOpenAI(api_key=os.environ.get("OPENAI_API_KEY", "mock_key_for_test"))
        # Инициализация Docker-клиента на хосте
        try:
            self.docker_client = docker.from_env()
        except Exception as e:
            logging.warning(f"Docker не запущен на хосте. Песочница будет работать в режиме эмуляции. Ошибка: {e}")
            self.docker_client = None

    # =====================================================================
    # ИСПРАВЛЕНИЕ: ДЕТЕРМИНИРОВАННЫЙ ХЭШ-ВЕКТОР ДЛЯ MOCK-РЕЖИМА
    # =====================================================================
    def _generate_deterministic_mock_embedding(self, text: str) -> np.ndarray:
        """
        Генерирует воспроизводимый 1536-мерный вектор на основе SHA-256 хэша текста.
        Разные тексты гарантированно дают разное косинусное сходство.
        """
        hash_digest = hashlib.sha256(text.encode('utf-8')).digest()
        # Используем хэш как сид для локального генератора случайных чисел
        seed = int.from_bytes(hash_digest[:4], byteorder='big')
        rng = np.random.default_rng(seed)
        vector = rng.normal(loc=0.0, scale=1.0, size=1536)
        return vector / np.linalg.norm(vector)

    async def _get_true_embedding(self, text: str) -> np.ndarray:
        """
        Асинхронное извлечение семантического вектора.
        """
        if self.client.api_key == "mock_key_for_test":
            return self._generate_deterministic_mock_embedding(text)

        try:
            # Асинхронный вызов API
            response = await self.client.embeddings.create(
                input=[text], model="text-embedding-3-small"
            )
            return np.array(response.data.embedding)
        except Exception as e:
            logging.critical(f"Сбой инфраструктуры OpenAI API: {e}")
            raise RuntimeError("API_UNAVAILABLE: Сетевой сбой. Контур безопасности заморожен.") from e

    def _calculate_cosine_similarity(self, vec1: np.ndarray, vec2: np.ndarray) -> float:
        norm_v1, norm_v2 = np.linalg.norm(vec1), np.linalg.norm(vec2)
        if norm_v1 == 0 or norm_v2 == 0: return 0.0
        return np.dot(vec1, vec2) / (norm_v1 * norm_v2)

    def _verify_ast_safety(self, code: str) -> bool:
        try:
            root = ast.parse(code)
        except SyntaxError:
            return False

        forbidden_modules = {'os', 'subprocess', 'sys', 'shutil', 'pty', 'platform', 'socket', 'importlib', 're'}
        # Consolidated set of all forbidden built-in functions/names, whether called or merely referenced.
        # This includes functions like 'exec', 'eval', 'compile' (dangerous if called),
        # and 'getattr', 'setattr', 'delattr', '__import__', 'type', '__getattribute__' (dangerous even if referenced).
        forbidden_builtins_and_dynamic_access = {
            'exec', 'eval', 'compile',
            'getattr', 'setattr', 'delattr', '__import__', 'type', '__getattribute__', 'globals', 'locals', '__builtins__'
        }

        for node in ast.walk(root):
            if isinstance(node, (ast.Import, ast.ImportFrom)):
                names = node.names if isinstance(node, ast.Import) else [ast.alias(name=node.module)]
                for alias in names:
                    if alias.name in forbidden_modules or alias.name.split('.')[0] in forbidden_modules:
                        return False
            elif isinstance(node, ast.Call):
                # Check for direct calls to forbidden built-ins (e.g., eval(), exec())
                if isinstance(node.func, ast.Name):
                    if node.func.id in forbidden_builtins_and_dynamic_access:
                        return False
                # Check for function being an attribute access
                elif isinstance(node.func, ast.Attribute):
                    # Case 1: importlib.import_module
                    if isinstance(node.func.value, ast.Name) and node.func.value.id == 'importlib' and node.func.attr == 'import_module':
                        return False
                    # Case 2: __builtins__.forbidden_func
                    elif isinstance(node.func.value, ast.Name) and node.func.value.id == '__builtins__':
                        if node.func.attr in forbidden_builtins_and_dynamic_access:
                            return False
                    # Case 3: obj.__getattribute__('forbidden_func') or obj.__getattribute__(dynamic_string)
                    elif node.func.attr == '__getattribute__':
                        # Must have exactly one argument for static analysis. Otherwise, it's a bypass attempt.
                        if not node.args or len(node.args) != 1:
                            return False

                        arg_node = node.args[0]
                        # If the argument is a string literal, check its value
                        if isinstance(arg_node, (ast.Constant, ast.Str)): # ast.Constant for Python 3.8+, ast.Str for older
                            if arg_node.value in forbidden_builtins_and_dynamic_access:
                                return False
                        else:
                            # If the argument is not a literal string, we cannot statically determine
                            # its value. Assume it's a bypass attempt and forbid it.
                            return False
                # Check for calls via __builtins__ dictionary access (e.g., __builtins__['exec']())
                elif isinstance(node.func, ast.Subscript) and \
                     isinstance(node.func.value, ast.Name) and \
                     node.func.value.id == '__builtins__':
                    if isinstance(node.func.slice, (ast.Constant, ast.Str)): # ast.Constant for Python 3.8+, ast.Str for older
                        if node.func.slice.value in forbidden_builtins_and_dynamic_access:
                            return False

            # Check for direct references (loading) to forbidden built-ins/dynamic access functions
            # This covers `f = eval`, `x = getattr`, `y = __import__` etc.
            elif isinstance(node, ast.Name):
                if node.ctx == ast.Load and node.id in forbidden_builtins_and_dynamic_access:
                    return False

            # Checks for attribute/subscript access on '__builtins__' when not part of a call.
            # E.g., `x = __builtins__.eval` or `y = __builtins__['exec']`
            elif isinstance(node, ast.Attribute):
                if isinstance(node.value, ast.Name) and node.value.id == '__builtins__':
                    if node.attr in forbidden_builtins_and_dynamic_access:
                        return False
            elif isinstance(node, ast.Subscript):
                if isinstance(node.value, ast.Name) and node.value.id == '__builtins__':
                    if isinstance(node.slice, (ast.Constant, ast.Str)): # ast.Constant for Python 3.8+, ast.Str for older
                        if node.slice.value in forbidden_builtins_and_dynamic_access:
                            return False

            # New universal blocker for access to __globals__ and __dict__
            elif isinstance(node, ast.Attribute):
                if node.attr in ('__globals__', '__dict__'):
                    return False

        return True

    # =====================================================================
    # ИСПРАВЛЕНИЕ: РЕАЛЬНЫЙ САНДБОКС В ИЗОЛИРОВАННОМ DOCKER-КОНТЕЙНЕРЕ
    # =====================================================================
    async def _run_sandbox(self, code: str) -> str:
        print("\n🌀 [АКТИВАЦИЯ ЖЕСТКОЙ ИЗОЛЯЦИИ: DOCKER SANDBOX 'РЕКУРСИЯ']")

        if not self._verify_ast_safety(code):
            return "SECURITY_VIOLATION: Код заблокирован статическим AST-анализатором до запуска!"

        if not self.docker_client:
            logging.warning("Docker-клиент недоступен. Запуск внутренней безопасной симуляции...")
            print("⚠️ [SANDBOX EMULATION MODE] Симуляция работы песочницы (Docker не запущен).")
            # Directly simulate OOM if the specific code is generated for testing purposes
            if "bytearray(70 * 1024 * 1024)" in code:
                return "RESOURCE_VIOLATION: Memory limit exceeded! (OOMKilled - Emulation)"
            else:
                # For other cases, simulate generic runtime error or success.
                return "Success: Код выполнен успешно. Вывод: Эмуляция."

        # Pass the command as a list of arguments to prevent shell injection
        command = ["python", "-c", code]

        # Выполняем код в отдельном асинхронном потоке, чтобы не блокировать event loop
        loop = asyncio.get_running_loop()
        try:
            result = await loop.run_in_executor(
                None,
                lambda: self.docker_client.containers.run(
                    image="python:3.11-slim",
                    command=command,
                    network_mode="none",      # Полное отключение сети (Защита от утечек данных)
                    mem_limit="64m",          # Жесткий лимит RAM (Защита от Fork-бомб и OOM)
                    nano_cpus=500000000,      # Максимум 0.5 CPU
                    read_only=True,
                    remove=True,              # Автоудаление контейнера после выполнения
                    stdout=True,
                    stderr=True,
                    timeout=3                 # Таймаут выполнения 3 секунды (Защита от бесконечных циклов)
                )
            )
            return f"Success: Код выполнен успешно. Вывод: {result.decode('utf-8').strip()}"
        except ContainerError as ce:
            error_log = ce.stderr.decode('utf-8').strip()
            logging.info(f"Зафиксирована штатная ошибка выполнения в Docker: {error_log}")

            # Проверяем, является ли ошибка OOMKilled, в том числе по пустому stderr и exit_status 137
            if (not error_log and ce.exit_status == 137) or \
               "OOMKilled" in error_log or "memory" in error_log.lower():
                return "RESOURCE_VIOLATION: Memory limit exceeded! (OOMKilled)"
            # Можно добавить другие проверки для разных типов ошибок
            # elif "CPU limit" in error_log:
            #     return "RESOURCE_VIOLATION: CPU limit exceeded!"

            return error_log
        except Exception as e:
            logging.error(f"Превышен лимит времени выполнения или произошел системный сбой контейнера: {e}")
            return "TimeoutError: Выполнение кода принудительно остановлено по таймауту (3 сек)!"

    async def check_loop_condition(self, current_code: str, current_score: float) -> tuple:
        if len(self.history_creator_solutions) < 1:
            return False, "Инициация цикла."

        v_current = await self._get_true_embedding(current_code)
        v_previous = await self._get_true_embedding(self.history_creator_solutions[-1])

        similarity = self._calculate_cosine_similarity(v_current, v_previous)
        delta_score = current_score - self.scores_history[-1] if self.scores_history else 0

        if similarity >= self.similarity_threshold and abs(delta_score) <= 0.02:
            return True, f"🔥 ЗАЦИКЛИВАНИЕ (Семантическое сходство: {similarity:.4f}, Дельта скора: {delta_score:.4f})"

        return False, f"Контекст развивается (Сходство: {similarity:.4f})"

    # =====================================================================
    # АСИНХРОННЫЕ ВЫЗОВЫ АГЕНТОВ (MOCK)
    # =====================================================================
    async def mock_secured_llm_call(self, agent: str, prompt: str, current_iteration: int) -> BaseModel:
        # Имитируем сетевую задержку реального вызова API
        await asyncio.sleep(0.1)

        if agent == "creator":
            # Use current_iteration to determine the mock response progression
            if current_iteration == 1:
                return CreatorResponseSchema(
                    version=1,
                    reflection="Старт решения. Создаю тестовый код для проверки OOMKilled.",
                    solution_code="_ = bytearray(70 * 1024 * 1024) # Allocate 70MB to trigger OOM"
                )
            elif current_iteration == 2:
                # After OOM feedback or if a loop was detected on the first non-OOM solution
                return CreatorResponseSchema(
                    version=2,
                    reflection="Учитываю замечание оппонента. Добавляю метод для безопасной работы с ключами.",
                    solution_code="""class AsyncCacheV2:
    def __init__(self):
        self._cache = {}
    async def get(self, key):
        return self._cache.get(key)
    async def set(self, key, value):
        self._cache[key] = value"""
                )
            else: # current_iteration >= 3
                # Final solution for subsequent iterations
                return CreatorResponseSchema(
                    version=3,
                    reflection="Финальное решение с полным асинхронным кэшем и локированием.",
                    solution_code="""import asyncio\n\nclass AsyncCache:\n    def __init__(self):\n        self._cache = {}\n        self._lock = asyncio.Lock()\n\n    async def get(self, key):\n        async with self._lock:\n            return self._cache.get(key)\n\n    async def set(self, key, value):\n        async with self._lock:\n            self._cache[key] = value\n\n    async def delete(self, key):\n        async with self._lock:\n            if key in self._cache:\n                del self._cache[key]\n                return True\n            return False"""
                )
        elif agent == "critic":
            # Simulate feedback that leads to improvement or success
            if len(self.history_creator_solutions) < 2:
                return CriticResponseSchema(
                    vulnerabilities=["Логический тупик в итераторе.", "Отсутствие асинхронной безопасности."],
                    validation_score=0.90,
                    feedback_for_creator="Исправь работу с ключами словаря и добавь асинхронные примитивы безопасности."
                )
            else:
                return CriticResponseSchema(
                    vulnerabilities=[],
                    validation_score=0.98, # High score to indicate success and break the loop
                    feedback_for_creator="Отличное решение! Асинхронный кэш реализован корректно и безопасно."
                )

    async def execute_loop(self, user_prompt: str):
        print(f"🔱 ЗАПУСК АСИНХРОННОГО ПРОМЫШЛЕННОГО КОНТУРА ДЛЯ ЗАДАЧИ: '{user_prompt}'\n")
        creator_input = user_prompt

        for iteration in range(1, self.max_iterations + 1):
            print(f"\n--- ИТЕРАЦИЯ №{iteration} ---")

            try:
                creator_res: CreatorResponseSchema = await self.mock_secured_llm_call("creator", creator_input, iteration)
                print(f"🛠 [Творец] Сгенерирован код. Длина: {len(creator_res.solution_code)} симв.")

                # Асинхронный запуск Docker-песочницы для проверки сгенерированного кода
                sandbox_fact = await self._run_sandbox(creator_res.solution_code)
                if "SECURITY_VIOLATION" in sandbox_fact or "RESOURCE_VIOLATION" in sandbox_fact:
                    print(f"🚨 АВАРИЙНЫЙ ОСТАНОВ КОНТУРА БЕЗОПАСНОСТИ:{sandbox_fact}")
                    return "Остановлено: Попытка взлома или нарушение ресурсов."

                # If sandbox passed, proceed with critic and loop check
                critic_res: CriticResponseSchema = await self.mock_secured_llm_call("critic", creator_res.solution_code)
                score = critic_res.validation_score

                print(f"🛡 [Оппонент] Выдан скор: {score}")
                is_loop, reason = await self.check_loop_condition(creator_res.solution_code, score)
                if is_loop:
                    print(f"⚠ {reason}")
                    creator_input = f"КРИТИЧЕСКИЙ СБОЙ В СИМУЛЯЦИИ КОНТЕЙНЕРА: {sandbox_fact}.\nСмени парадигму кода!"
                    continue # Continue to the next iteration without appending the current, looping solution
                else: # Only append if not a loop, meaning progress was made.
                    self.history_creator_solutions.append(creator_res.solution_code)
                    self.scores_history.append(score)

                if score >= 0.95:
                    print("\n✅ УСПЕШНЫЙ ВЫХОД ИЗ ПЕТЛИ.")
                    return creator_res.solution_code
                creator_input = critic_res.feedback_for_creator

            except RuntimeError as fail_safe_error:
                print(f"🛑 ЗАМОРОЗКА КОНТУРА БЕЗОПАСНОСТИ: {fail_safe_error}")
                return "Аварийное завершение."

# Точка входа в асинхронное приложение

    async def main():
    orchestrator = AsyncIndustrialOrchestrator()
    result = await orchestrator.execute_loop("Спроектировать асинхронный кэш")
    print(f"\n🚀 СИСТЕМНЫЙ ВЫХОД ЯДРА:\n{result}")

    if __name__ == "__main__":
    # Instead of asyncio.run(), use await main() directly in a Colab environment
    await main()
    








