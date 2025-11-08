# Движок классификации данных - Техническое описание

## Общая концепция

Движок классификации данных автоматически определяет уровень конфиденциальности данных перед загрузкой в аналитическое хранилище, применяет соответствующие меры защиты (обезличивание, маскирование, шифрование) и распределяет данные по зонам хранилища.

## Архитектурные принципы

### 1. Privacy by Design
- Классификация и защита встроены в pipeline
- Обезличивание по умолчанию для аналитики
- Минимизация данных на каждом этапе

### 2. Adaptability
- Автоматическое определение изменений схемы
- ML модели обучаются на новых данных
- Динамические политики классификации

### 3. Scalability
- Горизонтальное масштабирование компонентов
- Batch и stream processing
- Partitioning по уровням конфиденциальности

## Компоненты движка

### Ingestion Layer

#### Data Collector (Apache NiFi)
**Функции:**
- Подключение к источникам данных (REST API, JDBC, Files)
- Извлечение данных в batch или stream режиме
- Первичная валидация формата

**Настройка:**
```yaml
collectors:
  - name: emr_collector
    source: EMR Service API
    schedule: "*/15 * * * *"  # каждые 15 минут
    format: JSON
  - name: patient_collector
    source: Patient Service API
    schedule: "*/10 * * * *"
    format: JSON
```

#### Schema Detector
**Функции:**
- Автоматическое определение структуры данных
- Сравнение с предыдущими версиями схемы
- Обнаружение новых полей

**Алгоритм:**
```python
def detect_schema(data):
    schema = infer_schema(data)
    previous_schema = catalog.get_latest_schema(data_source)
    
    if schema != previous_schema:
        changes = diff_schemas(schema, previous_schema)
        notify_change_detector(changes)
        catalog.register_schema_version(schema)
    
    return schema
```

#### Change Detector (Debezium CDC)
**Функции:**
- Отслеживание изменений структуры таблиц
- Уведомление при появлении новых полей
- Триггер перенастройки классификатора

### Classification Layer

#### Content Scanner
**Функции:**
- Сканирование содержимого каждого поля
- Определение типа данных (текст, число, дата, и т.д.)
- Передача в соответствующие классификаторы

**Пример:**
```python
def scan_content(record):
    classified_fields = {}
    
    for field_name, value in record.items():
        field_type = detect_field_type(value)
        
        if field_type == 'text':
            classification = ml_classifier.classify(value)
        elif field_type == 'structured':
            classification = pattern_matcher.match(value)
        
        classified_fields[field_name] = classification
    
    return classified_fields
```

#### Pattern Matcher (Regex Engine)
**Функции:**
- Поиск ПД по регулярным выражениям
- Распознавание: телефоны, email, паспорта, СНИЛС, ИНН

**Паттерны:**
```python
PII_PATTERNS = {
    'phone': r'\+7\s?\(?\d{3}\)?\s?\d{3}[-\s]?\d{2}[-\s]?\d{2}',
    'email': r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}',
    'passport': r'\d{4}\s?\d{6}',
    'snils': r'\d{3}-\d{3}-\d{3}\s?\d{2}',
    'inn': r'\d{10}|\d{12}'
}
```

#### ML Classifier (TensorFlow)
**Функции:**
- Классификация неструктурированного текста
- Обнаружение медицинских терминов, диагнозов
- Определение чувствительности текста

**Модель:**
```python
# Предобученная BERT модель для классификации медицинского текста
model = load_model('medical_text_classifier')

def classify_text(text):
    # Токенизация
    tokens = tokenizer.encode(text)
    
    # Предсказание
    prediction = model.predict(tokens)
    
    # Категории: DIAGNOSIS, SYMPTOM, PRESCRIPTION, GENERAL
    category = argmax(prediction)
    confidence = max(prediction)
    
    if category in ['DIAGNOSIS', 'PRESCRIPTION'] and confidence > 0.8:
        return 'L4-PHI'
    elif category == 'SYMPTOM' and confidence > 0.7:
        return 'L3-PHI'
    else:
        return 'L2-MEDICAL'
```

#### Sensitivity Scorer
**Функции:**
- Агрегация результатов от Pattern Matcher и ML Classifier
- Присвоение итогового уровня конфиденциальности (L1-L4)
- Учет контекста (комбинация полей)

**Алгоритм:**
```python
def calculate_sensitivity(classified_fields):
    max_level = 'L1'
    
    for field, classification in classified_fields.items():
        # Паттерны ПД
        if classification['pattern'] in ['passport', 'snils']:
            max_level = max(max_level, 'L3')
        
        # ML классификация
        if classification['ml_category'] == 'L4-PHI':
            max_level = 'L4'
        
        # Комбинации (inference attack protection)
        if 'name' in classified_fields and 'birth_date' in classified_fields:
            max_level = max(max_level, 'L3')
    
    return max_level
```

#### Tag Manager
**Функции:**
- Присвоение тегов конфиденциальности
- Получение политик из Policy Store
- Применение политик (какие меры защиты нужны)

**Политики:**
```json
{
  "policies": [
    {
      "level": "L4",
      "type": "PHI",
      "measures": {
        "anonymization": "k-anonymity",
        "k": 10,
        "encryption": "AES-256",
        "access_control": "ABAC",
        "audit": "full"
      }
    },
    {
      "level": "L3",
      "type": "PII",
      "measures": {
        "masking": true,
        "tokenization": ["passport", "snils"],
        "encryption": "AES-256",
        "access_control": "RBAC"
      }
    }
  ]
}
```

### Anonymization Layer

#### Anonymizer (ARX Tool)
**Функции:**
- K-anonymity: группировка минимум k записей
- L-diversity: разнообразие в группах
- T-closeness: распределение чувствительных атрибутов
- Risk assessment

**Конфигурация:**
```python
anonymization_config = {
    'quasi_identifiers': ['age_range', 'gender', 'city'],
    'sensitive_attributes': ['diagnosis', 'treatment'],
    'k': 10,
    'l': 3,
    'suppression_limit': 0.05
}

anonymized_data = arx.anonymize(data, anonymization_config)
risk = arx.assess_risk(anonymized_data)

if risk['reidentification_risk'] > 0.01:
    # Увеличить k или применить больше generalization
    anonymization_config['k'] = 20
    anonymized_data = arx.anonymize(data, anonymization_config)
```

#### Data Masker
**Функции:**
- Маскирование ФИО ("Иван И******** И*********")
- Маскирование телефонов (+7 (***) ***-12-34)
- Частичное скрытие номеров

#### Tokenizer (Vault API)
**Функции:**
- Замена реальных значений на токены
- Обратимая токенизация через Vault
- Хранение mapping в защищенном хранилище

**Пример:**
```python
def tokenize_field(value, field_type):
    # Генерация токена
    token = vault.tokenize(value, field_type)
    
    # Vault хранит mapping: token -> encrypted_value
    return token

# Использование
passport_token = tokenize_field("1234 567890", "passport")
# Результат: "TKN_4f3d2e1a9b8c"
```

#### Pseudonymizer
**Функции:**
- UUID вместо ФИО
- Хеширование для уникальности
- Консистентная псевдонимизация (один ID для одного пациента)

### Storage Preparation Layer

#### Data Partitioner
**Функции:**
- Разделение данных по уровням конфиденциальности
- Маршрутизация в соответствующие зоны хранилища
- Метаданные для lineage

**Routing:**
```python
def partition_data(record, classification_level):
    if classification_level == 'L4':
        return {
            'raw_zone': encrypt(record),  # исходные зашифрованные
            'sensitive_zone': mask_pii(record),  # маскированные
            'anonymized_zone': anonymize(record)  # обезличенные
        }
    elif classification_level == 'L3':
        return {
            'raw_zone': encrypt(record),
            'sensitive_zone': tokenize(record),
            'anonymized_zone': pseudonymize(record)
        }
    elif classification_level in ['L1', 'L2']:
        return {
            'anonymized_zone': record,
            'public_zone': aggregate(record)
        }
```

#### Encryption Service
**Функции:**
- Шифрование перед загрузкой в RAW Zone
- Интеграция с Vault для получения ключей
- Шифрование на уровне записи (record-level)

#### Lineage Tracker
**Функции:**
- Регистрация источника данных
- Отслеживание трансформаций
- Запись в Data Catalog (Apache Atlas)

**Lineage example:**
```
EMR Service -> Data Collector -> Schema Detector -> 
Content Scanner -> ML Classifier -> Sensitivity Scorer (L4) -> 
Anonymizer (k=10) -> ANONYMIZED Zone
```

### Quality & Validation

#### Data Validator (Great Expectations)
**Функции:**
- Проверка качества после обезличивания
- Валидация: нет ли утечки ПД
- Проверка целостности данных

**Expectations:**
```python
expectations = [
    {
        'expectation': 'no_pii_in_anonymized',
        'check': 'regex_not_match',
        'pattern': PII_PATTERNS
    },
    {
        'expectation': 'k_anonymity_satisfied',
        'check': 'min_group_size',
        'k': 10
    },
    {
        'expectation': 'data_completeness',
        'check': 'not_null',
        'columns': ['patient_id', 'visit_date']
    }
]
```

#### Compliance Checker
**Функции:**
- Проверка соответствия политикам
- Проверка тегов классификации
- Блокировка загрузки при нарушениях

**Checks:**
```python
def check_compliance(record, tags):
    violations = []
    
    # Проверка: L4 данные обезличены?
    if 'L4' in tags and not is_anonymized(record):
        violations.append('L4 data not anonymized')
    
    # Проверка: зашифровано?
    if tags['classification'] in ['L3', 'L4'] and not is_encrypted(record):
        violations.append('Sensitive data not encrypted')
    
    # Проверка: есть ли audit trail?
    if not has_lineage(record):
        violations.append('Missing data lineage')
    
    if violations:
        raise ComplianceViolationError(violations)
    
    return True
```

## Зоны аналитического хранилища

### RAW Zone
**Назначение:** Исходные данные в оригинальном виде

**Характеристики:**
- Все данные зашифрованы (AES-256)
- Доступ только для Data Engineers
- Используется для reprocessing при изменении политик

### SENSITIVE Zone
**Назначение:** Данные L3-L4 с примененными мерами защиты

**Характеристики:**
- Маскирование или токенизация ПД
- RBAC: доступ только для авторизованных ролей
- Полный audit доступа
- Используется для детальной аналитики с ограничениями

### ANONYMIZED Zone
**Назначение:** Обезличенные данные для аналитики

**Характеристики:**
- K-anonymity (k >= 10)
- Нет прямых идентификаторов
- Доступ для аналитиков
- Используется для ML моделей, BI отчетов

### PUBLIC Zone
**Назначение:** Агрегированные данные для общего доступа

**Характеристики:**
- Только агрегаты (суммы, средние, count)
- Нет детальных записей
- Доступ для всех (RBAC: analyst, manager)
- Используется для dashboard, KPI

## Метрики эффективности классификации

### 1. Точность классификации (Accuracy)
```
Accuracy = (TP + TN) / (TP + TN + FP + FN)

TP: правильно классифицированные L4
TN: правильно классифицированные L1
FP: ложноположительные (L1 классифицирован как L4)
FN: ложноотрицательные (L4 классифицирован как L1)

Целевое значение: > 95%
```

### 2. Recall (Полнота) для L4
```
Recall_L4 = TP_L4 / (TP_L4 + FN_L4)

Критично: не пропустить конфиденциальные данные
Целевое значение: > 98%
```

### 3. Precision (Точность) для L4
```
Precision_L4 = TP_L4 / (TP_L4 + FP_L4)

Избыточная классификация приведет к ненужному обезличиванию
Целевое значение: > 90%
```

### 4. Throughput (Пропускная способность)
```
Throughput = количество обработанных записей / время

Целевое значение: > 10,000 записей/сек
```

### 5. Latency (Задержка)
```
Latency = время от получения данных до загрузки в DWH

Целевое значение: < 5 минут (batch), < 30 сек (stream)
```

### 6. Data Quality Score
```
Quality_Score = (1 - error_rate) * completeness * consistency

error_rate: доля ошибок классификации
completeness: доля полей с присвоенными тегами
consistency: консистентность классификации одинаковых данных

Целевое значение: > 0.95
```

### 7. Anonymization Risk
```
Risk = P(re-identification)

Измеряется через ARX risk assessment
Целевое значение: < 0.01 (1%)
```

### Dashboards для мониторинга

**Метрики в реальном времени:**
- Количество обработанных записей
- Распределение по уровням (L1/L2/L3/L4)
- Ошибки классификации
- Throughput и latency
- Utilization ресурсов

**Аналитические метрики:**
- Тренды изменения схем данных
- F1-score классификаторов
- ROC-AUC для ML моделей
- Compliance violations

## Scalability

### Горизонтальное масштабирование

**Content Scanner, Pattern Matcher, ML Classifier:**
- Stateless компоненты
- Можно добавлять инстансы по мере роста нагрузки
- Load balancing через Kubernetes

**Anonymizer:**
- Batch processing с параллелизацией
- Spark/Dask для больших объемов

**ClickHouse (DWH):**
- Кластер с репликацией и sharding
- Партиционирование по датам и уровням конфиденциальности

### При росте объема данных (5x)

**Текущий объем:** 100,000 записей/день
**Целевой объем:** 500,000 записей/день

**Масштабирование:**
1. Content Scanner: 1 -> 5 инстансов
2. ML Classifier: 1 -> 3 инстансов (GPU)
3. Anonymizer: batch размер 10k -> 50k, 4 parallel workers
4. ClickHouse: 1 node -> 3 node кластер

**Стоимость:** ~300,000 руб/мес (облако)

### При росте пользователей

**Текущие пользователи:** 15 врачей, 6 сотрудников, 3 аналитика
**Целевые пользователи:** 75 врачей, 30 сотрудников, 15 аналитиков

**Масштабирование DWH:**
- ANONYMIZED Zone: больше реплик для read queries
- Кэширование частых запросов (Redis)
- Pre-aggregated tables для dashboard

## Обновление и обучение

### ML модели

**Переобучение:**
- Раз в квартал на новых данных
- Добавление новых типов медицинских терминов
- Улучшение точности классификации

**Валидация:**
- A/B тестирование новых моделей
- Сравнение метрик с предыдущей версией
- Rollback при снижении точности

### Политики классификации

**Обновление:**
- При изменении законодательства (ФЗ-152)
- При добавлении новых типов данных
- При обнаружении новых рисков

**Версионирование:**
- Все политики версионируются
- Lineage содержит версию политики
- Возможность reprocessing со старыми политиками

## Выводы

Движок классификации обеспечивает:
1. Автоматическую защиту конфиденциальных данных
2. Адаптацию к изменениям структуры данных
3. Высокую точность классификации (>95%)
4. Масштабируемость до 5x роста
5. Соответствие Privacy by Design принципам
6. Полную прозрачность (Data Lineage)
7. Compliance с российским законодательством

