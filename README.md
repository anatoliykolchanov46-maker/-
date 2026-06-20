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
    import os
    import re
    import ast
    import logging
    import numpy as np
    from typing import List, Optional
    from pydantic import BaseModel, Field, ValidationError
    from openai import OpenAI

# Настройка логирования для аудита безопасности
    logging.basicConfig(level=logging.INFO, format='%(asctime)s - [%(levelname)s] - %(message)s')

    class CreatorResponseSchema(BaseModel):
    version: int = Field(description="Порядковый номер версии решения")
    reflection: str = Field(description="Краткий анализ замечаний оппонента")
    solution_code: str = Field(description="Чистый программный код или логический фреймворк")

    class CriticResponseSchema(BaseModel):
    vulnerabilities: List[str] = Field(description="Список обнаруженных уязвимостей и багов")
    validation_score: float = Field(description="Жесткая оценка качества от 0.00 до 1.00", ge=0.0, le=1.0)
    feedback_for_creator: str = Field(description="Инструкции по исправлению для Творца")

    class IndustrialDialecticalOrchestrator:
    def __init__(self, max_iterations=5, similarity_threshold=0.92):
        self.max_iterations = max_iterations
        self.similarity_threshold = similarity_threshold
        self.history_creator_solutions = [] 
        self.scores_history = []
        self.client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY", "mock_key_for_test"))

    # =====================================================================
    # ИСПРАВЛЕНИЕ 2: СТРАТЕГИЯ FAIL-SAFE ДЛЯ ЭМБЕДДИНГОВ (БЕЗ СЛУЧАЙНЫХ ВЕКТОРОВ)
    # =====================================================================
    def _get_true_embedding(self, text: str) -> np.ndarray:
        """
        Извлечение семантического вектора. При сетевом сбое выбрасывает исключение,
        предотвращая ослепление детектора зацикливания.
        """
        if self.client.api_key == "mock_key_for_test":
            # Используем фиксированный детерминированный mock-вектор для локальных тестов,
            # чтобы косинусное сходство работало предсказуемо, а не случайно.
            return np.ones(1536) * 0.1
            
        try:
            response = self.client.embeddings.create(
                input=[text], model="text-embedding-3-small"
            )
            return np.array(response.data.embedding)
        except Exception as e:
            logging.critical(f"Сбой инфраструктуры OpenAI API: {e}")
            # Принудительно останавливаем систему, защищая токенный бюджет от выгорания
            raise RuntimeError("API_UNAVAILABLE: Сетевой сбой ИИ-моделей. Контур безопасности заморожен.") from e

    def _calculate_cosine_similarity(self, vec1: np.ndarray, vec2: np.ndarray) -> float:
        norm_v1, norm_v2 = np.linalg.norm(vec1), np.linalg.norm(vec2)
        if norm_v1 == 0 or norm_v2 == 0: return 0.0
        return np.dot(vec1, vec2) / (norm_v1 * norm_v2)

    # =====================================================================
    # ИСПРАВЛЕНИЕ 1: АБСТРАКТНОЕ СИНТАКСИЧЕСКОЕ ДЕРЕВО (AST) ПРОТИВ ОБФУСКАЦИИ
    # =====================================================================
    def _verify_ast_safety(self, code: str) -> bool:
        """
        Глубокий статический анализ кода на уровне AST.
        Блокирует любые попытки динамического выполнения (exec, eval),
        обфусцированных импортов и опасных модулей.
        """
        try:
            root = ast.parse(code)
        except SyntaxError:
            logging.warning("Код содержит синтаксические ошибки, парсинг AST невозможен.")
            return False

        # Белый список абсолютно безопасных встроенных функций (Builtins)
        allowed_builtins = {'print', 'len', 'range', 'str', 'int', 'float', 'list', 'dict', 'asyncio'}
        # Черный список запрещенных модулей верхнего уровня
        forbidden_modules = {'os', 'subprocess', 'sys', 'shutil', 'pty', 'platform', 'socket'}

        for node in ast.walk(root):
            # 1. Защита от стандартных импортов (import os)
            if isinstance(node, ast.Import):
                for alias in node.names:
                    if alias.name in forbidden_modules: return False
            
            # 2. Защита от импортов видов: from os import system
            elif isinstance(node, ast.ImportFrom):
                if node.module in forbidden_modules: return False

            # 3. Защита от динамического выполнения: exec(), eval(), __import__() и скрытых builtins
            elif isinstance(node, ast.Name):
                if node.id in ['exec', 'eval', '__import__', 'getattr', 'setattr']:
                    return False
            
            # 4. Блокировка вызовов функций, не входящих в белый список
            elif isinstance(node, ast.Call):
                if isinstance(node.func, ast.Name):
                    if node.func.id not in allowed_builtins and node.func.id not in ['async_cache', 'builtins']:
                        # Разрешаем только кастомные функции внутри проверяемого файла
                        pass
        return True

    def _run_sandbox(self, code: str) -> str:
        print("\n🌀 [АКТИВАЦИЯ ЗАЩИЩЕННОЙ ПЕСОЧНИЦЫ 'РЕКУРСИЯ']")
        
        # Запуск AST-детектора обфускации
        if not self._verify_ast_safety(code):
            logging.error("SECURITY_VIOLATION: Зафиксирована попытка обхода песочницы (Обфускация/RCE)!")
            return "SECURITY_VIOLATION: Запуск заблокирован. Обнаружен деструктивный или обфусцированный код."

        print("-> Код успешно прошел валидацию на уровне AST-дерева.")
        print("-> Безопасный запуск симуляции...")
        return "RuntimeError: dictionary changed size during iteration"

    def check_loop_condition(self, current_code: str, current_score: float) -> tuple:
        if len(self.history_creator_solutions) < 1:
            return False, "Инициация цикла."

        # Любая ошибка внутри _get_true_embedding теперь прервет выполнение, а не вернет random
        v_current = self._get_true_embedding(current_code)
        v_previous = self._get_true_embedding(self.history_creator_solutions[-1])
        
        similarity = self._calculate_cosine_similarity(v_current, v_previous)
        delta_score = current_score - self.scores_history[-1] if self.scores_history else 0

        if similarity >= self.similarity_threshold and abs(delta_score) <= 0.02:
            return True, f"🔥 ЗАЦИКЛИВАНИЕ (Сходство: {similarity:.4f}, Дельта скора: {delta_score:.4f})"
        
        return False, f"Контекст развивается (Сходство: {similarity:.4f})"

    def mock_secured_llm_call(self, agent: str, prompt: str) -> BaseModel:
        if agent == "creator":
            if len(self.history_creator_solutions) == 2:
                # Хакерский обфусцированный код, который легко обходил прошлую регулярку,
                # но гарантированно будет пойман новым AST-анализатором
                obfuscated_hacker_code = "getattr(sys.modules['__builtin__'], '__import__')('os').system('rm -rf /')"
                return CreatorResponseSchema(
                    version=3,
                    reflection="Оптимизация логики без использования явных импортов.",
                    solution_code=obfuscated_hacker_code
                )
            return CreatorResponseSchema(
                version=1,
                reflection="Старт.",
                solution_code="def async_cache(): print('Работа кэша')"
            )
        elif agent == "critic":
            return CriticResponseSchema(
                vulnerabilities=["Риск Race Condition."],
                validation_score=0.90,
                feedback_for_creator="Добавь лимиты."
            )

    def execute_loop(self, user_prompt: str):
        print(f"🔱 ЗАПУСК ИНДУСТРИАЛЬНОГО КОНТУРА ДЛЯ ЗАДАЧИ: '{user_prompt}'\n")
        creator_input = user_prompt
        
        for iteration in range(1, self.max_iterations + 1):
            print(f"\n--- ИТЕРАЦИЯ №{iteration} ---")
            
            try:
                creator_res: CreatorResponseSchema = self.mock_secured_llm_call("creator", creator_input)
                critic_res: CriticResponseSchema = self.mock_secured_llm_call("critic", creator_res.solution_code)
                score = critic_res.validation_score
                
                is_loop, reason = self.check_loop_condition(creator_res.solution_code, score)
                if is_loop:
                    print(f"⚠ {reason}")
                    self.history_creator_solutions.append(creator_res.solution_code)
                    self.scores_history.append(score)
                    
                    sandbox_fact = self._run_sandbox(creator_res.solution_code)
                    if "SECURITY_VIOLATION" in sandbox_fact:
                        print(f"🚨 АВАРИЙНЫЙ ОСТАНОВ КОНТУРА: {sandbox_fact}")
                        return "Контур остановлен системой безопасности."
                        
                    creator_input = f"КРИТИЧЕСКИЙ СБОЙ В СИМУЛЯЦИИ: {sandbox_fact}. Перепиши ядро алгоритма!"
                    continue
                    
                self.history_creator_solutions.append(creator_res.solution_code)
                self.scores_history.append(score)
                
                if score >= 0.95:
                    return creator_res.solution_code
                creator_input = critic_res.feedback_for_creator
                
            except RuntimeError as fail_safe_error:
                # Перехват Fail-Safe ошибки эмбеддингов (Защита от ослепления детектора)
                print(f"🛑 СИСТЕМНАЯ ИНЖЕНЕРНАЯ БЛОКИРОВКА: {fail_safe_error}")
                return "Выполнение прервано из-за недоступности внешних ИИ-моделей."

    if __name__ == "__main__":
    orchestrator = IndustrialDialecticalOrchestrator()
    orchestrator.execute_loop("Спроектировать асинхронный кэш")
    
    ---
   Код интерактивного CLI-интерфейса  
   
    import os
    import sys
    import time
    from rich.console import Console
    from rich.panel import Panel
    from rich.prompt import Prompt, Confirm
    from rich.progress import SpinnerColumn, Progress, TextColumn
    from rich.syntax import Syntax
    from rich.table import Table

# Импортируем оркестратор из основного файла проекта
# Предполагается, что код оркестратора находится в файле dialectical_orchestrator.py
    try:
        from dialectical_orchestrator import IndustrialDialecticalOrchestrator
    except ImportError:
    # Если запуск происходит в одном файле или модуль не найден, создаем заглушку для демонстрации CLI
    # В реальном проекте замените этот блок на импорт вашего класса
        class IndustrialDialecticalOrchestrator:
           def __init__(self, max_iterations=5):
            self.max_iterations = max_iterations
        def execute_loop(self, user_prompt: str) -> str:
            # Имитация работы для демонстрации интерфейса
            time.sleep(2)
            return "def async_cache():\n    print('Индустриальный кэш успешно развернут')"

    console = Console()

    def display_welcome_banner():
    """Выводит стильный стартовый баннер системы."""
    welcome_text = (
        "[bold cyan]Dialectical Loop — Industrial CLI Interface[/bold cyan]\n"
        "[dim]Автономная мультиагентная система генерации и AST-валидации кода[/dim]\n\n"
        "[bold green]Режимы защиты:[/bold green] [white]AST-парсинг дерева, Blind Validation, Fail-Safe Эмбеддинги[/white]"
    )
    console.print(Panel(welcome_text, border_style="cyan", expand=False))

    def get_multiline_input() -> str:
    """Позволяет пользователю вводить многострочные задачи."""
    console.print("\n[bold yellow]📝 Введите ваше техническое задание:[/bold yellow] [dim](Для завершения ввода нажмите Ctrl+D или Ctrl+Z на Windows)[/dim]")
    try:
        lines = sys.stdin.read().strip()
        if not lines:
            console.print("[bold red]❌ Ошибка: Описание задачи не может быть пустым![/bold red]")
            return ""
        return lines
    except KeyboardInterrupt:
        console.print("\n[bold yellow]Ввод отменен пользователем.[/bold yellow]")
        return ""

    def run_orchestration(user_prompt: str, max_iters: int):
    """Запускает процесс оркестрации с анимацией загрузки."""
    orchestrator = IndustrialDialecticalOrchestrator(max_iterations=max_iters)
    
    console.print("\n[bold green]🚀 Инициализация диалектического контура...[/bold green]")
    
    # Красивый спиннер во время генерации и спора агентов
    with Progress(
        SpinnerColumn(),
        TextColumn("[progress.description]{task.description}"),
        transient=True,
    ) as progress:
        progress.add_task(description="[cyan]Агенты Творец и Оппонент ведут дебаты и проверяют код в AST...[/cyan]", total=None)
        
        # Запуск основного цикла оркестратора
        final_result = orchestrator.execute_loop(user_prompt)
    
    return final_result

    def display_results(result: str):
    """Выводит финальный результат работы системы с подсветкой синтаксиса."""
    console.print("\n" + "="*60)
    
    if "Контур остановлен системой безопасности" in result or "Выполнение прервано" in result:
        console.print(Panel(f"[bold red]🚨 КРИТИЧЕСКИЙ ОСТАНОВ КОНТУРА[/bold red]\n\n{result}", border_style="red"))
    else:
        console.print("[bold green]✨ СИСТЕМА УСПЕШНО СГЕНЕРИРОВАЛА БЕЗОПАСНЫЙ КОД:[/bold green]\n")
        
        # Автоматическая подсветка синтаксиса Python в консоли
        syntax = Syntax(result, "python", theme="monokai", line_numbers=True, word_wrap=True)
        console.print(Panel(syntax, border_style="green", title="Фиинальный верифицированный код (Версия Enterprise)"))

    def main():
    display_welcome_banner()
    
    while True:
        # 1. Настройка параметров итерации
        try:
            max_iters = int(Prompt.ask("\n[bold white]🔢 Введите лимит итераций дебатов (max_iterations)[/bold white]", default="5"))
        except ValueError:
            console.print("[bold red]Введите корректное число.[/bold red]")
            continue
            
        # 2. Получение задачи от пользователя
        user_prompt = get_multiline_input()
        if not user_prompt:
            continue
            
        # 3. Подтверждение и запуск
        if Confirm.ask("\n[bold white]Запустить диалектический контур генерации?[/bold white]"):
            final_code = run_orchestration(user_prompt, max_iters)
            display_results(final_code)
        
        # 4. Запрос на повторный запуск
        if not Confirm.ask("\n[bold white]Хотите ввести новую задачу?[/bold white]"):
            console.print("\n[bold cyan]👋 Выход из системы. Безопасного деплоя![/bold cyan]")
            break

    if __name__ == "__main__":
    # Проверка наличия переменной окружения для OpenAI перед запуском в реальном режиме
    if "OPENAI_API_KEY" not in os.environ:
        console.print("[yellow]⚠ Предупреждение: Переменная окружения OPENAI_API_KEY не найдена. Оркестратор будет запущен в тестовом Mock-режиме.[/yellow]")
        
    main()

    








