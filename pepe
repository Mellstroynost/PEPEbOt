import requests
import time
import random
import re
import json
import os
import sys
import traceback
from datetime import datetime, timedelta

# ============================================
# 📦 ЗАГРУЗКА ТОКЕНА ИЗ .ENV ФАЙЛА
# ============================================
try:
    from dotenv import load_dotenv
    # Пытаемся загрузить из .env
    env_path = os.path.join(os.path.dirname(__file__), '.env')
    if os.path.exists(env_path):
        load_dotenv(env_path)
        print(f"✅ Загружен .env файл: {env_path}")
    else:
        print(f"⚠️ .env файл не найден по пути: {env_path}")
except ImportError:
    print("⚠️ python-dotenv не установлен, использую переменные окружения")
    pass

# Получаем токен из переменной окружения
TOKEN = os.getenv('BOT_TOKEN')

if not TOKEN:
    print("❌ КРИТИЧЕСКАЯ ОШИБКА: Токен не найден!")
    print("\n🔧 ЧТО ДЕЛАТЬ:")
    print("1. Создайте файл .env в той же папке")
    print("2. Добавьте в него строку: BOT_TOKEN=ваш_токен_от_BotFather")
    print("3. Или установите переменную окружения BOT_TOKEN")
    sys.exit(1)

print(f"🔑 Токен загружен: {TOKEN[:8]}...{TOKEN[-4:]}")

BOT_USERNAME = "FLuxPR_bot"

# ============================================
# 🎯 ОСНОВНОЙ КЛАСС БОТА
# ============================================
class SelectiveReplyBot:
    def __init__(self, token, bot_username="FLuxPR_bot"):
        self.token = token
        self.bot_username = bot_username
        self.base_url = f"https://api.telegram.org/bot{token}"
        
        # 🔧 НАСТРОЙКИ АНТИСПАМА
        self.delete_user_links = True  # Удалять ссылки от пользователей
        self.allow_bot_links = True    # Разрешить боту отправлять ссылки (пасхалки)
        
        # Пути к файлам данных
        self.data_dir = os.path.join(os.path.dirname(__file__), 'data')
        self.ensure_data_dir()
        
        self.cake_data_file = os.path.join(self.data_dir, "cake_data.json")
        self.twist_data_file = os.path.join(self.data_dir, "twist_data.json")
        self.streak_file = os.path.join(self.data_dir, "streak_data.json")
        
        # Хранилище для команды /cake
        self.cake_data = self.load_cake_data()
        self.cake_cooldown = 20 * 60  # 20 минут в секундах
        
        # Хранилище для команды /twist (карточки)
        self.twist_data = self.load_twist_data()
        self.twist_charge_time = 1 * 60 * 60  # 1 час в секундах для зарядки
        
        # Хранилище для стриков
        self.streak_data = self.load_streak_data()
        
        # Цены продажи для каждой редкости
        self.sell_prices = {
            "обычная": 10,
            "редкий": 25,
            "эпический": 50,
            "мифический": 100,
            "легендарный": 250,
            "эксклюзив": 2500,
            "глич": 10000
        }
        
        # Тестовые айди для быстрого тестирования без таймера
        self.test_user_ids = ["7724617221", "7671086697"]
        
        # Специальная группа с измененными шансами
        self.special_group_id = "-5277344495"
        
        # Статистика работы
        self.start_time = time.time()
        self.error_count = 0
        self.message_count = 0
        self.max_errors = 100  # Максимальное количество ошибок до перезапуска
        
        # Кэш для быстрого доступа к ключевым словам
        self.setup_fast_triggers()
        
        # Получаем ID бота
        self.bot_id = self.get_bot_id()
        if not self.bot_id:
            print("❌ Не удалось получить ID бота!")
            return
        
        print(f"🤖 Бот: @{bot_username} (ID: {self.bot_id})")
        print(f"📁 Данные сохраняются в: {self.data_dir}")
        
    def ensure_data_dir(self):
        """Создает директорию для данных, если её нет"""
        try:
            if not os.path.exists(self.data_dir):
                os.makedirs(self.data_dir)
                print(f"✅ Создана директория для данных: {self.data_dir}")
        except Exception as e:
            print(f"⚠️ Ошибка создания директории: {e}")
    
    def setup_fast_triggers(self):
        """Настраивает быстрые триггеры для ускорения проверки"""
        self.my_words_set = {
            "што тепе", "привет", "пока", "как дела",
            "салам", "хай", "здарова", "прив", "передай привет"
        }
        
        self.bot_username_lower = self.bot_username.lower()
        self.commands_set = {
            "/start", f"/start@{self.bot_username_lower}",
            "/help", f"/help@{self.bot_username_lower}",
            "/cake", f"/cake@{self.bot_username_lower}",
            "/top", f"/top@{self.bot_username_lower}",
            "/topall", f"/topall@{self.bot_username_lower}",
            "/twist", f"/twist@{self.bot_username_lower}",
            "/inventory", f"/inventory@{self.bot_username_lower}",
            "/sell", f"/sell@{self.bot_username_lower}"
        }
        
        self.easter_eggs_set = {
            "мадара", "мут", "бойся", "мем", 
            "танцуй", "пепе шнене держи тортик"
        }
        
        self.keywords = {
            "пепе шнене", "передай привет"
        }.union(self.easter_eggs_set)
        
        self.role_commands_set = {
            "избить", "обнять", "поцеловать", "ударить", "погладить"
        }
    
    # ============================================
    # 💾 ЗАГРУЗКА И СОХРАНЕНИЕ ДАННЫХ
    # ============================================
    def load_json_data(self, file_path, default_value):
        """Универсальная загрузка JSON данных"""
        try:
            if os.path.exists(file_path):
                with open(file_path, 'r', encoding='utf-8') as f:
                    data = json.load(f)
                    print(f"✅ Загружено: {file_path}")
                    return data
            else:
                print(f"📝 Создан новый файл: {file_path}")
                return default_value
        except Exception as e:
            print(f"⚠️ Ошибка загрузки {file_path}: {e}")
            return default_value
    
    def save_json_data(self, file_path, data):
        """Универсальное сохранение JSON данных"""
        try:
            # Создаем временный файл для атомарной записи
            temp_file = file_path + '.tmp'
            with open(temp_file, 'w', encoding='utf-8') as f:
                json.dump(data, f, ensure_ascii=False, indent=2)
            # Заменяем оригинал
            os.replace(temp_file, file_path)
            return True
        except Exception as e:
            print(f"⚠️ Ошибка сохранения {file_path}: {e}")
            return False
    
    def load_cake_data(self):
        return self.load_json_data(self.cake_data_file, {})
    
    def save_cake_data(self):
        return self.save_json_data(self.cake_data_file, self.cake_data)
    
    def load_twist_data(self):
        return self.load_json_data(self.twist_data_file, {})
    
    def save_twist_data(self):
        return self.save_json_data(self.twist_data_file, self.twist_data)
    
    def load_streak_data(self):
        data = self.load_json_data(self.streak_file, {})
        # Конвертируем даты
        result = {}
        for user_id, d in data.items():
            try:
                result[user_id] = {
                    'current_streak': d.get('current_streak', 0),
                    'last_streak_update': datetime.fromisoformat(d['last_streak_update']).date() if d.get('last_streak_update') else None,
                    'daily_used': d.get('daily_used', False)
                }
            except:
                result[user_id] = {'current_streak': 0, 'last_streak_update': None, 'daily_used': False}
        return result
    
    def save_streak_data(self):
        save_data = {}
        for user_id, d in self.streak_data.items():
            save_data[user_id] = {
                'current_streak': d['current_streak'],
                'last_streak_update': d['last_streak_update'].isoformat() if d['last_streak_update'] else None,
                'daily_used': d.get('daily_used', False)
            }
        return self.save_json_data(self.streak_file, save_data)
    
    # ============================================
    # 🔧 ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ
    # ============================================
    def get_bot_id(self):
        result = self.api_request("getMe")
        if result and result.get("ok"):
            return result["result"]["id"]
        return None
    
    def is_message_from_bot(self, message):
        return "from" in message and message["from"].get("id") == self.bot_id
    
    def contains_user_links(self, text, from_bot=False):
        if not text or not self.delete_user_links:
            return False
        
        if from_bot and self.allow_bot_links:
            return False
        
        text_lower = text.lower()
        
        link_patterns = [
            r'https?://[^\s]+',
            r'www\.[^\s]+\.[a-z]{2,}',
            r't\.me/[^\s]+',
            r'telegram\.me/[^\s]+',
        ]
        
        allowed_patterns = [
            r'^/@[^\s]+$',
            r'/start@', '/help@', '/cake@', '/top@', '/topall@', '/twist@', '/inventory@', '/sell@'
        ]
        
        for allowed_pattern in allowed_patterns:
            if re.search(allowed_pattern, text_lower):
                return False
        
        for pattern in link_patterns:
            matches = re.findall(pattern, text_lower)
            for match in matches:
                is_command = False
                for cmd in ['/start', '/help', '/cake', '/top', '/topall', '/twist', '/inventory', '/sell']:
                    if match.startswith(f"{cmd}@"):
                        is_command = True
                        break
                if not is_command:
                    return True
        return False
    
    def delete_message(self, chat_id, message_id):
        data = {"chat_id": chat_id, "message_id": message_id}
        result = self.api_request("deleteMessage", data)
        return result and result.get("ok")
    
    def warn_user(self, chat_id, user_id, username, message_id=None):
        warning_text = (
            f"🚫 @{username}, кибербезопасность бота защищает чат от пиаров!\n"
            f"❌ Запрещено отправлять любые ссылки\n"
            f"✅ Пользуйтесь только текстовыми сообщениями"
        )
        data = {"chat_id": chat_id, "text": warning_text, "parse_mode": "HTML"}
        if message_id:
            data["reply_to_message_id"] = message_id
        result = self.api_request("sendMessage", data)
        return result and result.get("ok")
    
    def set_message_reaction(self, chat_id, message_id, reaction_types):
        data = {
            "chat_id": chat_id,
            "message_id": message_id,
            "reaction": [{"type": "emoji", "emoji": emoji} for emoji in reaction_types]
        }
        result = self.api_request("setMessageReaction", data)
        return result and result.get("ok")
    
    def is_reply_to_me(self, message):
        if "reply_to_message" not in message:
            return False, None
        reply_to = message["reply_to_message"]
        if reply_to.get("from", {}).get("id") == self.bot_id:
            return True, reply_to.get("text", "")
        return False, None
    
    def is_reply_to_any(self, message):
        return "reply_to_message" in message
    
    def should_respond(self, user_message, is_reply):
        if is_reply:
            return True
        
        user_msg_lower = user_message.lower()
        
        if "пепе шнене" in user_msg_lower:
            return True
        
        if user_msg_lower.startswith(('/', f'/start@{self.bot_username_lower}')):
            if user_msg_lower in self.commands_set or user_msg_lower.split()[0] in self.commands_set:
                return True
        
        if f"@{self.bot_username_lower}" in user_msg_lower:
            return True
        
        for keyword in self.easter_eggs_set:
            if keyword in user_msg_lower:
                return True
        
        for word in self.my_words_set:
            if word in user_msg_lower:
                return True
        
        return False
    
    # ============================================
    # 🎴 СИСТЕМА КАРТОЧЕК
    # ============================================
    def get_rarity_chance(self, chat_id=None):
        if chat_id and str(chat_id) == self.special_group_id:
            return "эксклюзив"
        
        rarities = {
            "обычная": 45,
            "редкий": 23,
            "эпический": 17,
            "мифический": 10,
            "легендарный": 4,
            "эксклюзив": 1,
        }
        
        roll = random.random() * 100
        cumulative = 0
        for rarity, chance in rarities.items():
            cumulative += chance
            if roll <= cumulative:
                return rarity
        return "обычная"
    
    def get_random_card(self, rarity):
        cards_database = {
            "обычная": [
                {"image": "https://i.yapx.ru/ck46R.jpg", "description": "типо деловой", "emoji": "⚪", "name": "Деловой блогер"},
                {"image": "https://i.yapx.ru/ck46w.jpg", "description": "тип крутой", "emoji": "⚪", "name": "Крутой блогер"},
                {"image": "https://i.yapx.ru/ck47y.jpg", "description": "дохуя ахуевший", "emoji": "⚪", "name": "Ахуевший блогер"},
                {"image": "https://i.yapx.ru/ck48T.jpg", "description": "понтовность", "emoji": "⚪", "name": "Понтовый блогер"}
            ],
            "редкий": [
                {"image": "https://i.yapx.ru/ck5g0.jpg", "description": "удивленый", "emoji": "🔵", "name": "Удивленный блогер"},
                {"image": "https://i.yapx.ru/ck5gg.jpg", "description": "лысый", "emoji": "🔵", "name": "Лысый блогер"},
                {"image": "https://i.yapx.ru/ck552.jpg", "description": "агрессивный", "emoji": "🔵", "name": "Агрессивный блогер"},
                {"image": "https://i.yapx.ru/ck56y.jpg", "description": "богатый", "emoji": "🔵", "name": "Богатый блогер"}
            ],
            "эпический": [
                {"image": "https://i.yapx.ru/ck6J6.jpg", "description": "умный", "emoji": "🟣", "name": "Умный блогер"},
                {"image": "https://i.yapx.ru/ck6K2.jpg", "description": "омайгадность", "emoji": "🟣", "name": "Омайгад блогер"},
                {"image": "https://i.yapx.ru/ck6Lf.jpg", "description": "не добрый", "emoji": "🟣", "name": "Не добрый блогер"}
            ],
            "мифический": [
                {"image": "https://i.yapx.ru/ck7ic.jpg", "description": "мемность", "emoji": "🟡", "name": "Мемный блогер"},
                {"image": "https://i.yapx.ru/ck7jU.jpg", "description": "типо кочок", "emoji": "🟡", "name": "Типо кочок блогер"},
                {"image": "https://i.yapx.ru/ck7oZ.jpg", "description": "я уже красный начальная форма", "emoji": "🟡", "name": "Красный блогер"},
                {"image": "https://i.yapx.ru/ck7rM.jpg", "description": "крутой 2 форма", "emoji": "🟡", "name": "Крутой 2 форма"}
            ],
            "легендарный": [
                {"image": "https://i.yapx.ru/ck7z8.jpg", "description": "фог мелл", "emoji": "🟠", "name": "Фог Мелл блогер"},
                {"image": "https://i.yapx.ru/ck71H.jpg", "description": "вторая стадия омайгадности", "emoji": "🟠", "name": "Омайгад 2 форма"},
                {"image": "https://i.yapx.ru/ck75P.jpg", "description": "я уже красный с шляпкой", "emoji": "🟠", "name": "Красный с шляпкой"}
            ],
            "эксклюзив": [
                {"image": "https://i.yapx.ru/ck8ZV.jpg", "description": "это уже не форма", "emoji": "🔴", "name": "Неформальный блогер"},
                {"image": "https://i.yapx.ru/ck8ci.jpg", "description": "омайгадность неизвестная форма", "emoji": "🔴", "name": "Неизвестная форма"}
            ]
        }
        
        cards = cards_database.get(rarity, cards_database["обычная"])
        return random.choice(cards)
    
    def get_rarity_emoji(self, rarity):
        emojis = {
            "обычная": "⚪", "редкий": "🔵", "эпический": "🟣",
            "мифический": "🟡", "легендарный": "🟠", "эксклюзив": "🔴", "глич": "🔮"
        }
        return emojis.get(rarity, "⚪")
    
    def update_twist_charges(self, user_id):
        user_id_str = str(user_id)
        current_time = time.time()
        
        if user_id_str in self.test_user_ids:
            if user_id_str not in self.twist_data:
                self.twist_data[user_id_str] = {
                    "total_twists": 0, "cards": {}, "available_twists": 3,
                    "last_update_time": current_time, "last_twist_time": 0,
                    "username": "", "last_username": "", "coins": 0, "inventory": []
                }
            else:
                self.twist_data[user_id_str]["available_twists"] = 3
                self.twist_data[user_id_str]["last_update_time"] = current_time
            return 3
        
        if user_id_str not in self.twist_data:
            self.twist_data[user_id_str] = {
                "total_twists": 0, "cards": {}, "available_twists": 0,
                "last_update_time": current_time, "last_twist_time": 0,
                "username": "", "last_username": "", "coins": 0, "inventory": []
            }
            return 0
        
        user_data = self.twist_data[user_id_str]
        if "last_update_time" not in user_data:
            user_data["last_update_time"] = current_time
        
        time_passed = current_time - user_data["last_update_time"]
        twists_earned = int(time_passed // self.twist_charge_time)
        
        if twists_earned > 0:
            time_used = twists_earned * self.twist_charge_time
            user_data["last_update_time"] += time_used
            available = user_data.get("available_twists", 0)
            user_data["available_twists"] = min(available + twists_earned, 3)
            self.save_twist_data()
            return user_data["available_twists"]
        
        return user_data.get("available_twists", 0)
    
    # ============================================
    # 📦 ОБРАБОТКА КОМАНД
    # ============================================
    def process_inventory_command(self, user_id, username):
        user_id_str = str(user_id)
        
        if user_id_str not in self.twist_data:
            return "📦 У вас пока нет карточек в инвентаре.\n🎴 Используйте /twist, чтобы получить первую карточку!"
        
        user_data = self.twist_data[user_id_str]
        inventory = user_data.get("inventory", [])
        coins = user_data.get("coins", 0)
        
        if not inventory:
            return f"📦 @{username}, ваш инвентарь пуст.\n🎴 Используйте /twist, чтобы получить карточки!\n💰 Ваши монеты: {coins}"
        
        message = f"📦 ИНВЕНТАРЬ @{username}\n\n💰 Монеты: {coins}\n📊 Всего карточек: {len(inventory)}\n\n"
        
        for i, card in enumerate(inventory[:10], 1):
            rarity = card.get("rarity", "обычная")
            name = card.get("name", "Без названия")
            emoji = self.get_rarity_emoji(rarity)
            price = self.sell_prices.get(rarity, 10)
            message += f"{i}. {emoji} {name} ({rarity}) - {price}💰\n"
        
        if len(inventory) > 10:
            message += f"\n📄 ... и еще {len(inventory) - 10} карточек\n"
        
        message += "\n💡 Продажа: /sell [номер]"
        return message
    
    def process_sell_command(self, user_id, username, text):
        user_id_str = str(user_id)
        
        if user_id_str not in self.twist_data:
            return "❌ У вас нет карточек для продажи."
        
        user_data = self.twist_data[user_id_str]
        inventory = user_data.get("inventory", [])
        
        if not inventory:
            return "❌ Ваш инвентарь пуст."
        
        parts = text.split()
        if len(parts) < 2:
            return "Используйте: /sell [номер]\nНапример: /sell 1"
        
        try:
            card_number = int(parts[1]) - 1
            if card_number < 0 or card_number >= len(inventory):
                return f"❌ Неверный номер. Всего карточек: {len(inventory)}"
        except ValueError:
            return "❌ Неверный формат."
        
        card = inventory.pop(card_number)
        rarity = card.get("rarity", "обычная")
        price = self.sell_prices.get(rarity, 10)
        
        user_data["coins"] = user_data.get("coins", 0) + price
        self.save_twist_data()
        
        return f"✅ Продано за {price}💰\n💰 Теперь у вас: {user_data['coins']} монет"
    
    def process_twist_command(self, user_id, username, chat_id=None):
        user_id_str = str(user_id)
        available = self.update_twist_charges(user_id)
        
        if available <= 0:
            return "🎴 Нет круток! Они копятся каждый час (макс. 3)"
        
        self.twist_data[user_id_str]["available_twists"] = available - 1
        rarity = self.get_rarity_chance(chat_id)
        card = self.get_random_card(rarity)
        
        if user_id_str not in self.twist_data:
            self.twist_data[user_id_str] = {
                "total_twists": 0, "cards": {}, "available_twists": available - 1,
                "last_update_time": time.time(), "last_twist_time": time.time(),
                "username": username, "last_username": username, "coins": 0, "inventory": []
            }
        else:
            self.twist_data[user_id_str]["last_twist_time"] = time.time()
            self.twist_data[user_id_str]["username"] = username
            if "inventory" not in self.twist_data[user_id_str]:
                self.twist_data[user_id_str]["inventory"] = []
            if "coins" not in self.twist_data[user_id_str]:
                self.twist_data[user_id_str]["coins"] = 0
        
        self.twist_data[user_id_str]["total_twists"] += 1
        
        card_data = {
            "id": f"{user_id_str}_{int(time.time())}_{random.randint(1000,9999)}",
            "rarity": rarity,
            "name": card.get("name", "Карточка"),
            "description": card["description"],
            "image": card["image"],
            "emoji": card["emoji"],
            "obtained_time": time.time()
        }
        self.twist_data[user_id_str]["inventory"].append(card_data)
        self.save_twist_data()
        
        emoji = self.get_rarity_emoji(rarity)
        caption = f"{emoji}━━━━━━━━━━{emoji}\n\n🏷️ {rarity.upper()}\n\n📝 {card['description']}\n\n💰 Цена: {self.sell_prices.get(rarity,10)} монет"
        
        if user_id_str in self.test_user_ids:
            caption += "\n\n🧪 ТЕСТОВЫЙ РЕЖИМ"
        
        if chat_id and str(chat_id) == self.special_group_id:
            caption += "\n\n✨ ЭКСКЛЮЗИВ!"
        
        return ("photo", card["image"], caption)
    
    def process_cake_command(self, user_id, username, chat_id=None):
        current_time = time.time()
        user_id_str = str(user_id)
        
        if user_id_str not in self.test_user_ids:
            if user_id_str in self.cake_data:
                last_used = self.cake_data[user_id_str]["last_used"]
                time_passed = current_time - last_used
                if time_passed < self.cake_cooldown:
                    left = int(self.cake_cooldown - time_passed)
                    m, s = divmod(left, 60)
                    return f"🍰 Жди еще {m}м {s}с"
        
        pieces = round(random.uniform(1.0, 5.0), 1)
        
        if user_id_str not in self.cake_data:
            self.cake_data[user_id_str] = {"total": 0.0, "last_used": current_time, "username": username}
        else:
            self.cake_data[user_id_str]["last_used"] = current_time
            self.cake_data[user_id_str]["username"] = username
        
        self.cake_data[user_id_str]["total"] += pieces
        self.save_cake_data()
        
        total = round(self.cake_data[user_id_str]["total"], 1)
        return f"🍰 @{username} съел {pieces} кусочков! Всего: {total}"
    
    def process_top_command(self, is_global=False):
        if not self.cake_data:
            return "🏆 Топ пуст"
        
        users = []
        for uid, data in self.cake_data.items():
            if data.get("total", 0) > 0:
                users.append({
                    "username": data.get("username", uid[:6]),
                    "total": data["total"]
                })
        
        users.sort(key=lambda x: x["total"], reverse=True)
        top = users[:20 if is_global else 10]
        
        title = "ГЛОБАЛЬНЫЙ ТОП" if is_global else "ТОП В ЧАТЕ"
        msg = f"🏆 {title}\n\n"
        medals = ["🥇", "🥈", "🥉"]
        
        for i, u in enumerate(top, 1):
            medal = medals[i-1] if i <= 3 else f"{i}."
            msg += f"{medal} @{u['username']} - {round(u['total'],1)}🍰\n"
        
        return msg
    
    # ============================================
    # 🎭 ПАСХАЛКИ И ОТВЕТЫ
    # ============================================
    def check_special_phrases(self, user_message):
        msg = user_message.lower()
        
        if "мадара" in msg:
            return ("animation", "https://media1.tenor.com/m/3AznEZyGnEoAAAAd/aizen-aizen-sosuke.gif", "")
        if "мут" in msg:
            return ("photo", "https://i.pinimg.com/736x/ac/88/50/ac88509d3b39f74d98062ca30f267a57.jpg", "")
        if "бойся" in msg and "ебл" in msg:
            return ("animation", "https://media1.tenor.com/m/C6KY0iG1su4AAAAd/%D0%B1%D0%BE%D0%B9%D1%81%D1%8F%D0%B5%D0%B1%D0%BB%D0%B8.gif", "")
        if "пепе шнене танцуй" in msg:
            return ("animation", "https://media1.tenor.com/m/cu-fNvjNyVoAAAAC/tsukasa-dance.gif", "")
        if "пепе шнене держи тортик" in msg:
            return ("animation", "https://media1.tenor.com/m/XQHivKfXRS8AAAAC/cake-eat.gif", "")
        return None
    
    def generate_hello_response(self, username, user_message):
        msg = user_message.lower()
        
        if "передай привет яган дон" in msg:
            return "я гандон"
        
        text = ""
        if "передай привет" in msg:
            parts = user_message.split("передай привет", 1)
            text = parts[1].strip() if len(parts) > 1 else "катакбасику"
        
        responses = [
            f"приветик {text}",
            f"здарово {text}",
            f"поревет {text}",
            f"хаваю {text}",
            f"передаю привет {text}"
        ]
        return random.choice(responses)
    
    def execute_my_code(self, user_message, user_id, username, is_reply=False, chat_id=None, message_id=None):
        msg_lower = user_message.lower()
        bot_lower = self.bot_username.lower()
        
        # Команды
        if msg_lower in ["/inventory", f"/inventory@{bot_lower}"]:
            return self.process_inventory_command(user_id, username)
        
        if msg_lower.startswith("/sell") or msg_lower.startswith(f"/sell@{bot_lower}"):
            return self.process_sell_command(user_id, username, user_message)
        
        if msg_lower in ["/twist", f"/twist@{bot_lower}"]:
            self.update_twist_charges(user_id)
            return self.process_twist_command(user_id, username, chat_id)
        
        if msg_lower in ["/cake", f"/cake@{bot_lower}"]:
            return self.process_cake_command(user_id, username, chat_id)
        
        if msg_lower in ["/top", f"/top@{bot_lower}"]:
            return self.process_top_command(False)
        
        if msg_lower in ["/topall", f"/topall@{bot_lower}"]:
            return self.process_top_command(True)
        
        if msg_lower in ["/start", f"/start@{bot_lower}"]:
            return f"пливет пакушай тортика\n🎴 /twist - карточки\n🍰 /cake - тортик\n📦 /inventory - инвентарь"
        
        if msg_lower in ["/help", f"/help@{bot_lower}"]:
            return "📋 Команды:\n/cake - тортик (20м)\n/twist - карточки (1ч)\n/inventory - инвентарь\n/sell - продать\n/top - топ\n/topall - глобальный топ"
        
        # Пасхалки
        special = self.check_special_phrases(user_message)
        if special:
            return special
        
        if "передай привет" in msg_lower:
            return self.generate_hello_response(username, user_message)
        
        # Ответы на reply
        if is_reply:
            if "пепе шнене дай тортик" in msg_lower:
                return "если флукс такую функцию добавит"
            if "люблю" in msg_lower:
                return "и я тепа"
            if "пепе шнене обними меня" in msg_lower:
                self.set_message_reaction(chat_id, message_id, ["💘"])
                return "срадостью"
            if "пепе шнене хороший мальчик" in msg_lower or "пепе шнене крутой" in msg_lower:
                self.set_message_reaction(chat_id, message_id, ["💘"])
                return "взаимно"
            
            responses = [
                "ни пищи дибил",
                "я твой отец",
                "соси",
                "пащол нахуи",
                "жры мьенчик"
            ]
            return random.choice(responses)
        
        # Обычные ответы
        if "пепе шнене" in msg_lower:
            return "пр"
        if f"@{self.bot_username}" in user_message:
            return "хули ти менья атмещаеш"
        
        return None
    
    # ============================================
    # 🌐 API ЗАПРОСЫ
    # ============================================
    def api_request(self, method, data=None, retries=3):
        url = f"{self.base_url}/{method}"
        
        for attempt in range(retries):
            try:
                response = requests.post(url, json=data, timeout=30)
                if response.status_code == 200:
                    return response.json()
                elif response.status_code == 429:
                    time.sleep(60)
                else:
                    print(f"⚠️ API {response.status_code}")
                    return None
            except requests.exceptions.Timeout:
                print(f"⚠️ Таймаут {method}")
                time.sleep(5)
            except Exception as e:
                print(f"⚠️ Ошибка API: {e}")
                time.sleep(5)
        return None
    
    def send_media(self, chat_id, media_type, url, caption="", reply_to_message_id=None):
        data = {"chat_id": chat_id, media_type: url}
        if caption:
            data["caption"] = caption
        if reply_to_message_id:
            data["reply_to_message_id"] = reply_to_message_id
        
        method = "sendAnimation" if media_type == "animation" else "sendPhoto"
        result = self.api_request(method, data)
        return result and result.get("ok")
    
    def send_reply(self, chat_id, text, reply_to_message_id):
        data = {
            "chat_id": chat_id,
            "text": text,
            "reply_to_message_id": reply_to_message_id,
            "parse_mode": "HTML"
        }
        return self.api_request("sendMessage", data)
    
    # ============================================
    # 🔄 ОБРАБОТКА СООБЩЕНИЙ
    # ============================================
    def process_message(self, message):
        try:
            chat_id = message["chat"]["id"]
            user_id = message["from"]["id"]
            username = message["from"].get("first_name", "друг")
            message_id = message["message_id"]
            
            # Приветствие новых участников
            if "new_chat_members" in message:
                for new_member in message["new_chat_members"]:
                    if new_member["id"] != self.bot_id:
                        self.send_media(
                            chat_id=chat_id,
                            media_type="animation",
                            url="https://media1.tenor.com/m/1YnAZm61NWYAAAAC/%D0%BD%D0%BE%D0%B2%D0%B5%D0%BD%D1%8C%D0%BA%D0%B8%D0%B9-%D0%B7%27%D1%97%D0%B1%D0%B0%D0%B2%D1%81%D1%8F-%D0%B7-%D1%87%D0%B0%D1%82%D1%83.gif",
                            caption="",
                            reply_to_message_id=message_id
                        )
            
            # Проверка на ссылки
            is_from_bot = self.is_message_from_bot(message)
            
            if "text" in message:
                user_message = message["text"]
                self.message_count += 1
                
                if self.message_count % 50 == 0:
                    print(f"📊 Сообщений: {self.message_count}")
                
                if not is_from_bot and self.delete_user_links:
                    if self.contains_user_links(user_message, False):
                        self.delete_message(chat_id, message_id)
                        self.warn_user(chat_id, user_id, username, message_id)
                        return
                
                is_reply = self.is_reply_to_me(message)[0]
                
                if self.should_respond(user_message, is_reply):
                    response = self.execute_my_code(
                        user_message, user_id, username, is_reply, chat_id, message_id
                    )
                    
                    if response is None:
                        return
                    elif isinstance(response, tuple) and len(response) == 3:
                        media_type, url, caption = response
                        self.send_media(chat_id, media_type, url, caption, message_id)
                    elif isinstance(response, str):
                        self.send_reply(chat_id, response, message_id)
        
        except Exception as e:
            print(f"⚠️ Ошибка: {e}")
            self.error_count += 1
    
    # ============================================
    # 🚀 ЗАПУСК
    # ============================================
    def run(self):
        if not self.bot_id:
            print("❌ Не могу запустить бота")
            return
        
        print("\n" + "="*60)
        print("🎯 БОТ ЗАПУЩЕН!")
        print("="*60)
        print(f"🤖 @{self.bot_username}")
        print(f"📁 Данные: {self.data_dir}")
        print(f"🍰 Тортики: {len(self.cake_data)} юзеров")
        print(f"🎴 Карточки: {len(self.twist_data)} юзеров")
        print("="*60)
        
        last_update_id = 0
        
        while True:
            try:
                if self.error_count > self.max_errors:
                    print("⚠️ Слишком много ошибок, перезапуск...")
                    self.error_count = 0
                    self.cake_data = self.load_cake_data()
                    self.twist_data = self.load_twist_data()
                
                updates = self.api_request("getUpdates", {
                    "offset": last_update_id + 1,
                    "timeout": 30
                })
                
                if updates and updates.get("ok"):
                    for update in updates["result"]:
                        last_update_id = update["update_id"]
                        if "message" in update:
                            self.process_message(update["message"])
                    
                    if self.message_count % 100 == 0:
                        self.save_cake_data()
                        self.save_twist_data()
                
                time.sleep(1)
                
            except KeyboardInterrupt:
                print("\n🛑 Остановка...")
                self.save_cake_data()
                self.save_twist_data()
                break
            except Exception as e:
                print(f"⚠️ Крит. ошибка: {e}")
                self.error_count += 1
                time.sleep(30)


# ============================================
# 🚀 ТОЧКА ВХОДА
# ============================================
if __name__ == "__main__":
    print("="*60)
    print("🚀 ЗАПУСК БОТА НА PYTHONANYWHERE")
    print("="*60)
    
    # Проверяем наличие токена
    if not TOKEN:
        print("\n❌ ТОКЕН НЕ НАЙДЕН!")
        print("\n🔧 Решение:")
        print("1. Создайте файл .env в папке с ботом:")
        print("   nano .env")
        print("2. Добавьте строку:")
        print("   BOT_TOKEN=8540823367:AAFHUcU5oLN40rikYFiK_6fK6IT8L-LoSHw")
        print("3. Сохраните и запустите снова")
        sys.exit(1)
    
    # Бесконечный цикл с авто-перезапуском
    while True:
        try:
            bot = SelectiveReplyBot(TOKEN, BOT_USERNAME)
            bot.run()
            print("🔄 Перезапуск через 10 сек...")
            time.sleep(10)
        except Exception as e:
            print(f"⚠️ Фатальная ошибка: {e}")
            print("🔄 Перезапуск через 30 сек...")
            time.sleep(30)
