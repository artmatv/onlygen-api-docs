# OnlyGen Widget — Интеграция в CRM

Встраиваемый виджет для генерации жестов прямо из панели оператора.

## Быстрый старт

```html
<iframe
  src="https://onlygen.mvt-soft.work/static/widget/index.html?key=YOUR_API_KEY"
  width="450"
  height="700"
  frameborder="0"
  allow="clipboard-write"
></iframe>
```

## Параметры URL

| Параметр | Обязательный | Значение | По умолчанию |
|----------|-------------|----------|-------------|
| `key` | Да | API ключ (`gc_...`) | — |
| `theme` | Нет | `dark` / `light` | `light` |
| `engine` | Нет | `seedream5` / `seedream4` | `seedream5` |
| `lang` | Нет | `ru` / `en` | `ru` |

### Пример с тёмной темой

```html
<iframe
  src="https://onlygen.mvt-soft.work/static/widget/index.html?key=gc_xxx&theme=dark&lang=ru"
  width="450"
  height="700"
  frameborder="0"
></iframe>
```

## Интеграция с CRM через postMessage

Виджет отправляет события в родительское окно через `window.postMessage`. Это позволяет CRM автоматически получать результат генерации.

### События

**Успешная генерация:**
```js
{
  type: 'onlygen_result',
  task_id: 'abc123...',
  result_url: 'https://...result_image.png',
  status: 'completed'
}
```

**Ошибка:**
```js
{
  type: 'onlygen_error',
  error: 'Rate limit exceeded'
}
```

### Пример обработки в CRM

```js
window.addEventListener('message', (event) => {
  if (event.data.type === 'onlygen_result') {
    // Вставить картинку в чат с фаном
    insertImageToChat(event.data.result_url);
  }

  if (event.data.type === 'onlygen_error') {
    showNotification('Ошибка генерации: ' + event.data.error);
  }
});
```

## Как получить API ключ

Запросите ключ у администратора. Ключ имеет формат `gc_...` и срок действия 30 дней (продлевается).

## Движки

| Движок | ID | Описание |
|--------|----|----------|
| **Seedream 5.0 Lite** | `seedream5` | Умный, стабильная идентичность |
| **Seedream 4.5 Edit** | `seedream4` | Максимальный реализм и детализация |

Оба движка поддерживают:
- Готовые пресеты жестов (34 шт.)
- Свой промпт (текстовое описание)
- Свой референс (фото-образец жеста)

## Возможности виджета

- Выбор движка генерации
- Категории пресетов с горизонтальным скроллом
- Свой промпт или свой референс-фото
- Drag & drop загрузка фото
- Повторная генерация (кнопка «Заново»)
- Скачивание и копирование URL результата
- Светлая и тёмная тема
- Русский и английский язык

## Требования

- Браузер: Chrome 80+, Firefox 78+, Safari 14+, Edge 80+
- Размер iframe: минимум 360x600px, рекомендуется 450x700px
- API ключ должен быть активен
