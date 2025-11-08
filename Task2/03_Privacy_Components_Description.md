# Описание компонентов Privacy by Design

## Privacy Layer - центральный компонент защиты

### 1. Encryption Service
**Назначение:** Централизованное шифрование/дешифрование всех конфиденциальных данных

**Функции:**
- Шифрование данных перед сохранением (AES-256-GCM)
- Дешифрование при авторизованном доступе
- Интеграция с KMS для управления ключами
- Токенизация чувствительных полей

**Privacy by Design:**
- Privacy as Default: все данные L3+ шифруются автоматически
- End-to-End Security: шифрование на всех этапах

### 2. Anonymization Service
**Назначение:** Обезличивание данных для вторичного использования

**Функции:**
- K-anonymity (группировка минимум k записей)
- L-diversity (разнообразие в группах)
- Differential Privacy (добавление шума)
- Генерализация (точные значения -> диапазоны)
- Подавление уникальных значений

**Privacy by Design:**
- Data Minimization: минимум данных для аналитики
- Purpose Limitation: разные уровни обезличивания для разных целей

### 3. Access Control Service
**Назначение:** Контроль доступа на основе ролей и атрибутов

**Функции:**
- RBAC для базовых ролей
- ABAC для сложных правил (врач -> только свои пациенты)
- Policy Engine (OPA) для динамических политик
- Break-the-glass для экстренных случаев

**Privacy by Design:**
- Least Privilege: минимальные необходимые права
- Need-to-Know: доступ только к релевантным данным

### 4. Audit Service
**Назначение:** Полное логирование всех операций с конфиденциальными данными

**Функции:**
- Логирование каждого доступа к ПД
- Who, What, When, Why
- Non-repudiation (невозможность отказа)
- Real-time алерты при аномалиях
- Интеграция с SIEM

**Privacy by Design:**
- Visibility and Transparency: полная прозрачность операций
- Accountability: ответственность за действия

### 5. Consent Management
**Назначение:** Управление согласиями пациентов на обработку ПД

**Функции:**
- Цифровой реестр согласий
- Версионирование согласий
- Механизм отзыва согласия
- Проверка согласия перед обработкой
- Гранулярные согласия (на разные цели)

**Privacy by Design:**
- Respect for User Privacy: права пациента на первом месте
- User Control: пациент контролирует свои данные

### 6. DSAR Service
**Назначение:** Автоматизация прав субъектов данных (Data Subject Access Request)

**Функции:**
- Портал для подачи запросов
- Автоматический поиск данных по всем системам
- Генерация отчета о хранимых данных
- Процесс удаления (Right to be Forgotten)
- SLA 30 дней

**Privacy by Design:**
- Respect for User Privacy: реализация прав ФЗ-152 ст.14
- Transparency: пациент видит все свои данные

## Analytics Layer с Privacy

### 7. ETL Pipeline с обезличиванием
**Назначение:** Загрузка данных в аналитическую систему с автоматическим обезличиванием

**Процесс:**
```
Исходные данные (EMR)
    ↓
Извлечение (Extract)
    ↓
Классификация по тегам
    ↓
Обезличивание (L4 -> L2)
  - Удаление прямых идентификаторов
  - K-anonymity
  - Генерализация
    ↓
Загрузка в Analytics DB
```

**Privacy by Design:**
- Privacy Embedded: обезличивание встроено в pipeline
- Purpose Limitation: аналитика работает только с необходимым уровнем данных

### 8. Data Catalog с Data Lineage
**Назначение:** Каталогизация данных, отслеживание происхождения и использования

**Функции:**
- Автоматическая регистрация всех датасетов
- Теги конфиденциальности
- Data Lineage: от источника до использования
- Impact analysis: что изменится при модификации
- Поиск данных по меткам

**Privacy by Design:**
- Visibility: понимание, где какие данные
- Accountability: отслеживание цепочки обработки

### 9. Data Anonymizer (ARX)
**Назначение:** Специализированный движок обезличивания

**Методы:**
- K-anonymity с настраиваемым k
- L-diversity для предотвращения homogeneity атак
- T-closeness для дополнительной защиты
- Risk assessment обезличенных данных

**Параметры:**
- Для BI: k=10, l=3
- Для ML: k=5, differential privacy ε=0.1
- Для публикации: k=20, l=5, t-closeness

## Новые компоненты в архитектуре

### Key Management Service (KMS)
- Централизованное хранилище ключей
- Автоматическая ротация ключей
- HSM для критичных ключей
- Audit всех операций с ключами

### API Gateway с Security
- Аутентификация (JWT)
- Авторизация (проверка токенов)
- Rate limiting
- Audit всех API calls
- DDoS protection

### SIEM Integration
- Централизованный сбор логов
- Корреляция событий
- Автоматические алерты
- Incident response triggers

## Принципы взаимодействия

### Запрос данных пациента врачом

```
Врач -> API Gateway
  ↓
API Gateway -> Access Control: проверить права
  ↓
Access Control -> Consent Mgmt: проверить согласие
  ↓
Access Control -> EMR Service: разрешен доступ
  ↓
EMR Service -> Encryption Service: дешифровать
  ↓
Encryption Service -> KMS: получить ключ
  ↓
EMR Service -> Primary DB: прочитать
  ↓
Параллельно:
  All Services -> Audit Service: залогировать
  ↓
Audit Service -> Audit DB + SIEM
```

### Загрузка данных в аналитику

```
Scheduled Job -> ETL Pipeline
  ↓
ETL -> EMR Service: извлечь данные
  ↓
ETL -> Data Catalog: зарегистрировать источник
  ↓
ETL -> Anonymizer: обезличить (k=10)
  ↓
Anonymizer -> Analytics DB: загрузить
  ↓
ETL -> Data Catalog: зарегистрировать lineage
```

## Обеспечение Privacy by Design принципов

| Принцип | Как реализовано |
|---------|----------------|
| 1. Proactive not Reactive | Encryption/Access Control встроены, не добавлены после |
| 2. Privacy as Default | Все данные L3+ шифруются автоматически |
| 3. Privacy Embedded | Privacy Layer встроен в архитектуру |
| 4. Full Functionality | Система функциональна с защитой |
| 5. End-to-End Security | От фронтенда до БД все защищено |
| 6. Visibility | Audit, Data Catalog, Lineage |
| 7. Respect for User | DSAR, Consent Management, портал |

## Масштабируемость

- Все сервисы Privacy Layer могут масштабироваться горизонтально
- Encryption Service: stateless, можно добавлять инстансы
- Anonymization: batch processing, параллелизация
- Audit Service: async обработка, буферизация
- Analytics Platform: ClickHouse кластер для роста данных

## Соответствие законодательству

- ФЗ-152: реализованы технические меры защиты, права субъектов
- ФЗ-323: врачебная тайна через ABAC и audit
- Приказ ФСТЭК №21: регистрация событий, контроль целостности
- ISO 27001: системный подход к ИБ

