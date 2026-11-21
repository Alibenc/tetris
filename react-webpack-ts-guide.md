# React + Webpack + TypeScript с нуля

**Задача:** самостоятельно настроить сборку React-приложения без Vite и Create React App.

## 1. Инициализация

```bash
mkdir react-webpack-ts
cd react-webpack-ts
npm init -y
```

Установи зависимости:

```bash
npm i react react-dom
npm i -D typescript @types/react @types/react-dom webpack webpack-cli webpack-dev-server ts-loader html-webpack-plugin css-loader style-loader
```

## 2. Структура проекта

```text
react-webpack-ts/
├── public/
│   └── index.html
├── src/
│   ├── App.tsx
│   ├── index.tsx
│   └── styles.css
├── webpack.config.cjs
├── tsconfig.json
└── package.json
```

## 3. Настройка TypeScript

Создай `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["DOM", "DOM.Iterable", "ES2020"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noEmit": true
  },
  "include": ["src"]
}
```

## 4. Настройка Webpack

Создай `webpack.config.cjs`:

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = (_env, argv) => ({
  mode: argv.mode || 'development',
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
      { test: /\.tsx?$/, use: 'ts-loader', exclude: /node_modules/ },
      { test: /\.css$/, use: ['style-loader', 'css-loader'] },
    ],
  },
  plugins: [
    new HtmlWebpackPlugin({ template: './public/index.html' }),
  ],
  devServer: {
    port: 3000,
    hot: true,
    open: true,
  },
});
```

Обрати внимание на `entry`, `output`, `resolve`, `module.rules` и `plugins`: разберись, за что отвечает каждый параметр.

## 5. Создание React-приложения

`public/index.html`:

```html
<!doctype html>
<html lang="ru">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>React + Webpack + TS</title>
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
import './styles.css';

const root = document.getElementById('root');
if (!root) throw new Error('Root element not found');

createRoot(root).render(<App />);
```

`src/App.tsx`:

```tsx
import { useState } from 'react';

export default function App() {
  const [count, setCount] = useState(0);

  return (
    <main>
      <h1>React + Webpack + TypeScript</h1>
      <button onClick={() => setCount(count + 1)}>
        Нажатий: {count}
      </button>
    </main>
  );
}
```

`src/styles.css`:

```css
body {
  font-family: Arial, sans-serif;
  margin: 40px;
}
```

## 6. Скрипты запуска

Добавь в `package.json` в поле `scripts`:

```json
{
  "start": "webpack serve --mode development",
  "build": "webpack --mode production",
  "typecheck": "tsc --noEmit"
}
```

Запуск:

```bash
npm start
```

Production-сборка и проверка типов:

```bash
npm run build
npm run typecheck
```

Webpack создаст папку `dist` при сборке. Её содержимое — готовые статические файлы приложения.

## 7. Самостоятельная практика

1. Добавь компонент `Counter`, вынеси в него счётчик и типизируй его props.
2. Подключи SCSS (`sass`, `sass-loader`) и замени `styles.css` на `styles.scss`.
3. Настрой импорт SVG и PNG через Webpack 5 Asset Modules.
4. Разберись, чем отличаются `development` и `production`, и сравни сборки.

**Критерий готовности:** приложение открывается через `npm start`, счётчик работает, `npm run typecheck` и `npm run build` завершаются без ошибок.
