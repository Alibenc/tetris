# React + TypeScript + Webpack + Tailwind CSS

Собери проект **с нуля, без Vite и Create React App**. Используем Webpack 5 и Tailwind CSS 4.

## 1. Инициализация

```bash
mkdir react-webpack-tailwind
cd react-webpack-tailwind
npm init -y
npm install react react-dom
npm install -D typescript @types/react @types/react-dom webpack webpack-cli webpack-dev-server ts-loader html-webpack-plugin style-loader css-loader postcss postcss-loader tailwindcss @tailwindcss/postcss
```

## 2. Структура проекта

```text
react-webpack-tailwind/
├── public/
│   └── index.html
├── src/
│   ├── App.tsx
│   ├── index.tsx
│   └── index.css
├── postcss.config.cjs
├── tsconfig.json
├── webpack.config.cjs
└── package.json
```

## 3. TypeScript

Создай `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "Node",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noEmit": false
  },
  "include": ["src"]
}
```

> Важно: для `ts-loader` не ставь `noEmit: true`, иначе TypeScript не отдаст JavaScript Webpack.

## 4. Настройка Webpack

Создай `webpack.config.cjs`:

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
  entry: './src/index.tsx',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js',
    clean: true,
  },
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
  },
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/,
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader', 'postcss-loader'],
      },
    ],
  },
  plugins: [
    new HtmlWebpackPlugin({ template: './public/index.html' }),
  ],
  devServer: {
    port: 3000,
    hot: true,
  },
};
```

## 5. Подключение Tailwind CSS

Создай `postcss.config.cjs`:

```js
module.exports = {
  plugins: {
    '@tailwindcss/postcss': {},
  },
};
```

В `src/index.css`:

```css
@import "tailwindcss";
```

Для Tailwind 4 не нужны `tailwind.config.js`, команда `tailwindcss init` и директивы `@tailwind base/components/utilities`.

## 6. React

`public/index.html`:

```html
<!doctype html>
<html lang="ru">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>React + Webpack</title>
  </head>
  <body>
    <div id="root"></div>
  </body>
</html>
```

`src/index.tsx`:

```tsx
import { createRoot } from 'react-dom/client';
import App from './App';
import './index.css';

createRoot(document.getElementById('root')!).render(<App />);
```

`src/App.tsx`:

```tsx
export default function App() {
  return (
    <main className="flex min-h-screen items-center justify-center bg-slate-950">
      <div className="rounded-xl bg-slate-800 p-8 text-center text-white shadow-lg">
        <h1 className="text-3xl font-bold">React + TS + Tailwind</h1>
        <button className="mt-5 rounded-lg bg-blue-600 px-5 py-2 hover:bg-blue-500">
          Работает!
        </button>
      </div>
    </main>
  );
}
```

## 7. Команды запуска

Добавь в `package.json` в раздел `scripts`:

```json
"scripts": {
  "start": "webpack serve --mode development",
  "build": "webpack --mode production",
  "typecheck": "tsc --noEmit"
}
```

```bash
npm run start      # http://localhost:3000
npm run typecheck  # проверка TypeScript
npm run build      # production-сборка в dist/
```

## 8. Небольшая практика

Сделай **карточку пользователя** с аватаром, именем, описанием, тегами навыков и кнопкой подписки.

- Вынеси карточку в компонент `ProfileCard.tsx`.
- Опиши `interface Profile` и передавай данные через props.
- Оформи карточку только классами Tailwind.
- Сделай адаптивность для телефона и hover-эффекты.

**Готово, когда:** сайт запускается, стили Tailwind применяются, `npm run typecheck` и `npm run build` проходят без ошибок.

Документация: [Webpack + TypeScript](https://webpack.js.org/guides/typescript/) · [Tailwind + PostCSS](https://tailwindcss.com/docs/installation/using-postcss)
