# Механизм тегирования данных

## Система классификации и тегов

### Уровни конфиденциальности

```
[L4-CRITICAL] - Специальная категория ПД (медицинские данные)
[L3-HIGH]     - Обычные ПД (ФИО, паспорт, контакты)
[L2-MEDIUM]   - Внутренние данные (финансы, операционные)
[L1-LOW]      - Общедоступные
```

### Типы данных

```
[PII]         - Personally Identifiable Information
[PHI]         - Protected Health Information
[FIN]         - Финансовые данные
[AUTH]        - Аутентификационные данные
[OPS]         - Операционные данные
```

### Статус обработки

```
[RAW]         - Исходные данные
[MASKED]      - Обфусцированные
[TOKENIZED]   - Токенизированные
[ANONYMIZED]  - Обезличенные
[ENCRYPTED]   - Зашифрованные
```

## Схема тегирования

### Формат тега

```
[LEVEL]-[TYPE]-[STATUS]-[JURISDICTION]
```

**Примеры:**
- `[L4-PHI-ENCRYPTED-RU]` - Критические медданные, зашифрованные, РФ юрисдикция
- `[L3-PII-MASKED-RU]` - ПД, маскированные
- `[L2-FIN-RAW-RU]` - Финансовые данные, исходные

### Метаданные для файлов

```json
{
  "classification": "L4-PHI",
  "status": "ENCRYPTED",
  "encryption": {
    "algorithm": "AES-256-GCM",
    "key_id": "key-medical-2024"
  },
  "access_control": {
    "roles": ["doctor"],
    "attributes": ["assigned_patient"]
  },
  "retention": {
    "min_years": 25,
    "policy": "medical_records"
  },
  "jurisdiction": "RU",
  "compliance": ["FZ-152", "FZ-323"],
  "audit_required": true,
  "created": "2024-01-15T10:30:00Z",
  "owner": "doctor_uuid_123"
}
```

## Автоматическое тегирование

### Правила классификации

**По содержимому (Content-based):**
- Регулярные выражения для обнаружения ПД (телефоны, email, паспорта)
- ML-модели для определения медицинских терминов
- Словари конфиденциальных терминов

**По контексту (Context-based):**
- Местоположение файла (папка медкарт -> L4-PHI)
- Автор документа (врач создал -> потенциально PHI)
- Связи с другими данными

**По метаданным:**
- Тип файла (медицинская форма -> PHI)
- Название файла (содержит "диагноз" -> L4)

### Движок тегирования

```
Входящий файл/данные
      ↓
[Сканер содержимого]
  - Regex для ПД
  - NLP для медицинских терминов
  - Структурный анализ
      ↓
[Классификатор]
  - Определение уровня L1-L4
  - Определение типа (PII/PHI/FIN)
      ↓
[Применение политик]
  - Автоматическое шифрование для L3+
  - Настройка прав доступа
  - Включение аудита
      ↓
[Присвоение тегов]
  - Запись метаданных
  - Обновление каталога данных
      ↓
Сохранение с тегами
```

## Каталог данных (Data Catalog)

### Структура реестра

```sql
CREATE TABLE data_catalog (
  id UUID PRIMARY KEY,
  file_path VARCHAR(500),
  classification VARCHAR(20),  -- L1/L2/L3/L4
  data_type VARCHAR(50),       -- PII/PHI/FIN/etc
  status VARCHAR(20),          -- RAW/ENCRYPTED/etc
  encryption_key_id VARCHAR(100),
  owner_id UUID,
  created_at TIMESTAMP,
  last_accessed TIMESTAMP,
  access_count INT,
  tags JSONB,
  retention_policy VARCHAR(50),
  audit_required BOOLEAN
);

CREATE INDEX idx_classification ON data_catalog(classification);
CREATE INDEX idx_data_type ON data_catalog(data_type);
```

## Политики на основе тегов

### Tag-Based Policies

```yaml
policies:
  - name: "L4-PHI-policy"
    match:
      classification: "L4"
      data_type: "PHI"
    enforce:
      encryption: "AES-256"
      access_control: "ABAC"
      audit: "full"
      retention_min: "25 years"
      backup_encryption: true
      dlp: true
      
  - name: "L3-PII-policy"
    match:
      classification: "L3"
      data_type: "PII"
    enforce:
      encryption: "AES-256"
      access_control: "RBAC"
      audit: "full"
      masking: true
      
  - name: "L2-policy"
    match:
      classification: "L2"
    enforce:
      access_control: "RBAC"
      audit: "basic"
      encryption: "optional"
```

## Инструменты тегирования

### Рекомендуемые решения

1. **Microsoft Purview (Azure Information Protection)**
   - Автоматическая классификация
   - Встроенные политики для GDPR/ФЗ-152
   - Интеграция с Office 365

2. **Apache Atlas**
   - Open-source data catalog
   - Метаданные и lineage
   - Классификация и тегирование

3. **Collibra Data Intelligence Cloud**
   - Data governance platform
   - Автоматическая классификация
   - Policy enforcement

4. **Custom solution**
   - Python + ML для классификации
   - PostgreSQL для каталога
   - REST API для интеграции

## Жизненный цикл тегов

```
Создание данных
    ↓
Автоматическое тегирование
    ↓
Применение политик
    ↓
Периодический пересмотр (review)
    ↓
Обновление тегов при изменении
    ↓
Архивация с сохранением тегов
    ↓
Удаление согласно retention policy
```

## Аудит тегирования

- Логирование всех изменений тегов
- Кто изменил, когда, почему
- Алерты при снижении уровня классификации
- Регулярный аудит корректности тегов

## Интеграция с Data Lineage

Теги должны передаваться по цепочке:
- Исходный файл [L4-PHI-RAW] 
- → Обработка → [L4-PHI-PROCESSING]
- → Результат → [L4-PHI-ENCRYPTED]
- → Аналитика → [L2-ANONYMIZED]

Отслеживание происхождения данных через теги.

