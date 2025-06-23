# BotManager - Система управления Telegram ботами в реальном времени

## Описание

BotManager - это сервис для управления всеми Telegram ботами клиентов с возможностью автоматического обновления токенов в реальном времени при изменениях в базе данных.

## Основные функции

### 🔄 Автоматическое отслеживание изменений
- **MongoDB Change Streams** - прослушивание изменений в коллекции `customers`
- **Мгновенное обновление** токенов ботов при изменении в БД
- **Кеширование** активных ботов в памяти для быстрого доступа

### 🤖 Управление ботами
- **Автоматическая инициализация** всех ботов при запуске сервера
- **Валидация токенов** при добавлении/обновлении
- **Статусы ботов**: `active`, `inactive`, `error`
- **Отслеживание состояния** каждого бота

### 📡 Event-driven архитектура
- **События** для уведомлений о изменениях ботов
- **Логирование** всех операций с эмодзи
- **Обработка ошибок** с детализацией

## Архитектура

```typescript
BotManager
├── bots: Map<customerId, BotInstance>    // Кеш активных ботов
├── changeStream: MongoDB Change Stream    // Прослушивание изменений
└── EventEmitter                          // События для уведомлений
```

### BotInstance
```typescript
interface BotInstance {
    bot: Telegraf;              // Экземпляр Telegram бота
    customerId: string;         // ID клиента
    username: string;           // Имя клиента
    token: string;              // Токен бота
    status: 'active' | 'inactive' | 'error';
    lastUpdated: Date;          // Время последнего обновления
}
```

## События

```typescript
// Бот добавлен
botManager.on('bot:added', (data) => {
    console.log(`🤖 Bot added: ${data.username} (@${data.botUsername})`);
});

// Бот обновлен
botManager.on('bot:updated', (data) => {
    console.log(`🔄 Bot updated: ${data.username} (@${data.botUsername})`);
});

// Бот удален
botManager.on('bot:removed', (data) => {
    console.log(`🗑️ Bot removed: ${data.username}`);
});

// Ошибка бота
botManager.on('bot:error', (data) => {
    console.log(`❌ Bot error for ${data.username}:`, data.error);
});

// Ошибка Change Stream
botManager.on('stream:error', (data) => {
    console.error('❌ Change stream error:', data.error);
});
```

## API методы

### Основные методы
```typescript
// Инициализация сервиса
await botManager.initialize();

// Получить бота по customerId
const bot = botManager.getBot(customerId);

// Получить информацию о боте
const botInfo = botManager.getBotInfo(customerId);

// Отправить сообщение
const result = await botManager.sendMessage(customerId, chatId, message);

// Проверить статус бота
const status = await botManager.checkBotStatus(customerId);

// Получить статистику
const stats = botManager.getStats();

// Перезагрузить всех ботов
await botManager.reload();

// Остановить сервис  
await botManager.stop();
```

### Статистика
```typescript
const stats = botManager.getStats();
/*
{
    total: 5,           // Всего ботов
    active: 3,          // Активных
    inactive: 0,        // Неактивных  
    error: 2,           // С ошибками
    isWatching: true    // Change Stream активен
}
*/
```

## REST API эндпоинты

### Для админа
```http
GET /api/messages/bot-manager-stats
Authorization: Bearer <admin_jwt_token>
```

Ответ:
```json
{
    "message": "Bot manager statistics", 
    "stats": {
        "total": 5,
        "active": 3,
        "inactive": 0, 
        "error": 2,
        "isWatching": true
    },
    "bots": [
        {
            "customerId": "...",
            "username": "client1", 
            "status": "active",
            "lastUpdated": "2024-01-15T10:30:00.000Z"
        }
    ],
    "timestamp": "2024-01-15T10:35:00.000Z"
}
```

## Преимущества новой системы

### ⚡ Производительность
- **Кеширование** - боты загружаются один раз и хранятся в памяти
- **Нет повторной инициализации** - каждый запрос использует готовый экземпляр
- **Быстрая отправка** сообщений без создания новых соединений

### 🔄 Актуальность данных  
- **Мгновенное обновление** при изменении токена в БД
- **Автоматическая синхронизация** без перезапуска сервера
- **Change Streams** отслеживают все операции: insert, update, delete

### 🛡️ Надежность
- **Обработка ошибок** с детализацией
- **Статусы ботов** для диагностики проблем
- **Восстановление соединений** при сбоях Change Stream
- **Graceful shutdown** с корректным закрытием соединений

### 📊 Мониторинг
- **Подробное логирование** всех операций
- **События** для интеграции с системами мониторинга  
- **Статистика** использования ботов
- **Диагностика** проблемных ботов

## Использование в коде

### До (старый подход)
```typescript
// Каждый раз создавался новый экземпляр
const bot = new Telegraf(token);
await bot.telegram.sendMessage(chatId, message);
```

### После (с BotManager)
```typescript
// Используем готовый экземпляр из кеша
const result = await botManager.sendMessage(customerId, chatId, message);
```

## Запуск и настройка

1. **Автоматический запуск** при старте сервера в `src/index.ts`
2. **Инициализация** после подключения к MongoDB
3. **Слушание событий** для логирования
4. **Graceful shutdown** при остановке приложения

## Мониторинг изменений в реальном времени

Система автоматически отслеживает:

### Добавление нового клиента
```javascript
// В MongoDB
db.customers.insertOne({
    username: "new_client",
    botToken: "7234567890:AAH...",
    password: "generated123"
});

// BotManager автоматически:
// 1. Получит уведомление через Change Stream  
// 2. Создаст новый экземпляр бота
// 3. Добавит в кеш
// 4. Выдаст событие 'bot:added'
```

### Обновление токена
```javascript
// В MongoDB  
db.customers.updateOne(
    { username: "existing_client" },
    { $set: { botToken: "7234567890:NEW_TOKEN..." } }
);

// BotManager автоматически:
// 1. Получит уведомление об изменении
// 2. Создаст новый экземпляр с новым токеном
// 3. Заменит старый в кеше
// 4. Выдаст событие 'bot:updated'
```

### Удаление клиента
```javascript
// В MongoDB
db.customers.deleteOne({ username: "old_client" });

// BotManager автоматически:
// 1. Получит уведомление об удалении
// 2. Удалит бота из кеша  
// 3. Выдаст событие 'bot:removed'
```

## Логи в действии

```bash
🔄 Initializing BotManager...
✅ Bot added for customer: client1 (@client1_bot)
✅ Bot added for customer: client2 (@client2_bot) 
❌ Failed to add bot for customer client3: 401 Unauthorized
✅ BotManager initialized with 2 bots
👁️ Started watching database changes

📡 Database change detected: update
🔄 Bot updated for customer: client1 (@client1_new_bot)

📡 Database change detected: insert  
✅ Bot added for customer: client4 (@client4_bot)
```

Этот подход обеспечивает максимальную производительность и актуальность данных без необходимости перезапуска сервера! 🚀 