# React + TypeScript + Webpack + Tailwind CSS + Ant Design

Собери проект **с нуля, без Vite и Create React App**. Используем Webpack 5, Tailwind CSS 4 и готовые компоненты Ant Design.

## 1. Инициализация

```bash
mkdir react-webpack-tailwind
cd react-webpack-tailwind
npm init -y
npm install react react-dom antd
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

## 6. React + Ant Design

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

`src/App.tsx` — пример: `Card`, `Input`, `Button` и `Tag` из Ant Design, а расположение и фон — через Tailwind:

```tsx
import { useState } from 'react';
import { Button, Card, Input, Tag } from 'antd';

export default function App() {
  const [name, setName] = useState('');
  const [savedName, setSavedName] = useState('');

  return (
    <main className="flex min-h-screen items-center justify-center bg-slate-100 p-6">
      <Card title="Профиль" className="w-full max-w-md shadow-lg">
        <div className="flex flex-col gap-4">
          <Input
            placeholder="Введите имя"
            value={name}
            onChange={(event) => setName(event.target.value)}
          />
          <Button type="primary" onClick={() => setSavedName(name.trim())}>
            Сохранить
          </Button>
          {savedName && <Tag color="blue">Привет, {savedName}!</Tag>}
        </div>
      </Card>
    </main>
  );
}
```

Ant Design уже содержит стили компонентов: отдельно подключать `antd.css` не нужно. Классы `flex`, `p-6`, `max-w-md` и другие предоставляет Tailwind. `@types/antd` также не нужен — типы включены в библиотеку.

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
- Используй `Card`, `Button`, `Input` и `Tag` из Ant Design, а для компоновки — классы Tailwind.
- Сделай адаптивность для телефона и hover-эффекты.

**Готово, когда:** сайт запускается, стили Tailwind применяются, `npm run typecheck` и `npm run build` проходят без ошибок.

Документация: [Webpack + TypeScript](https://webpack.js.org/guides/typescript/) · [Tailwind + PostCSS](https://tailwindcss.com/docs/installation/using-postcss) · [Ant Design](https://ant.design/components/overview/)
