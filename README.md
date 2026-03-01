<div align="center">

```
██╗  ██╗ █████╗  ██████╗ ███████╗ ██╗██╗   ██╗
██║  ██║██╔══██╗██╔═══██╗██╔════╝███║╚██╗ ██╔╝
███████║███████║██║   ██║███████╗ ╚██║ ╚████╔╝ 
██╔══██║██╔══██║██║   ██║╚════██║  ██║  ╚██╔╝  
██║  ██║██║  ██║╚██████╔╝███████║  ██║   ██║   
╚═╝  ╚═╝╚═╝  ╚═╝ ╚═════╝ ╚══════╝  ╚═╝   ╚═╝  
```

**haos1y.site** — личный сайт-визитка

*reverse engineering developer · musician*

[![GitHub Pages](https://img.shields.io/badge/Deployed-GitHub%20Pages-222?logo=github&logoColor=white&labelColor=0a0a12)](https://haos1y.github.io/haos1y.site/)
[![Svelte](https://img.shields.io/badge/Svelte-4-FF3E00?logo=svelte&logoColor=white)](https://svelte.dev/)
[![Vite](https://img.shields.io/badge/Vite-5-646CFF?logo=vite&logoColor=white)](https://vitejs.dev/)
[![Three.js](https://img.shields.io/badge/Three.js-r183-black?logo=threedotjs&logoColor=white)](https://threejs.org/)

</div>

---

## ✨ Фичи

| Фича | Описание |
|------|----------|
| 🖥️ **macOS-интерфейс** | Оконная система в стиле macOS с traffic lights, drag & drop, minimize/close |
| 🎵 **Аудиоплеер** | Кастомный плеер с 3D-визуализатором спектра и анимацией |
| 🎯 **Aim Trainer** | Полноценная 3D-игра для тренировки аима на базе Three.js |
| ⚡ **Загрузочный экран** | Двухфазная загрузка с прогресс-баром и анимацией cursor-hint |
| 🌟 **Частицы** | Интерактивные частицы, реагирующие на движение мыши |
| 🔍 **Spotlight-поиск** | Поиск в стиле macOS Spotlight |
| 🖱️ **Кастомный курсор** | Свой курсор с анимацией hover-эффектов |
| 🌙 **Тёмная/светлая тема** | Переключение темы с плавной анимацией |
| 📱 **Адаптивность** | Корректное отображение на всех устройствах |

---

## 🎯 Aim Trainer

Встроенная FPS-игра для тренировки аима — без серверов, полностью в браузере.

- **3D-пистолет** с анимацией отдачи и вспышкой дула
- **Трассеры пуль** с затуханием
- **2 режима:** Static (неподвижные цели) и Moving (по синусоиде)
- **Комбо-система** с множителем очков до 5x
- **Таблица лидеров** (Топ 10, хранится в localStorage)
- **Никнеймы** — запоминаются при первом входе
- **Glassmorphism UI** в стиле остального сайта

---

## 🛠️ Стек

```
Svelte 4       — UI-фреймворк
Vite 5         — сборщик
Three.js r183  — 3D-рендеринг (Aim Trainer)
GitHub Actions — CI/CD
GitHub Pages   — хостинг
```

---

## 🚀 Запуск локально

```bash
# Клонировать репозиторий
git clone https://github.com/haos1y/haos1y.site.git
cd haos1y.site

# Установить зависимости
npm install

# Запустить dev-сервер
npm run dev
```

Открой [`http://localhost:5173/haos1y.site/`](http://localhost:5173/haos1y.site/) в браузере.

### Другие команды

```bash
npm run build    # Собрать продакшн-бандл
npm run preview  # Предпросмотр продакшн-сборки
```

---

## 📦 Деплой

Сайт автоматически собирается и деплоится на **GitHub Pages** при каждом пуше в ветку `main` через **GitHub Actions**.

```
push → main → GitHub Actions (vite build) → gh-pages branch → haos1y.site
```

---

## 📁 Структура

```
mysitebio/
├── public/             # Статические файлы (иконки, шрифты, музыка)
├── src/
│   ├── App.svelte      # Главный компонент (окно, плеер, dock, частицы)
│   ├── AimTrainer.svelte  # 3D Aim Trainer (Three.js)
│   ├── LoadingScreen.svelte  # Двухфазная загрузка
│   └── main.js         # Точка входа
├── index.html
├── vite.config.js
└── package.json
```

---

<div align="center">

made with 🖤 by **haos1y**

</div>
