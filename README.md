# visual-diff

**visual-diff** — библиотека для строгой проверки визуального соответствия двух отрендеренных интерфейсов.

Проект сравнивает две версии UI не только по скриншоту, но и по структуре DOM и вычисленным стилям. Итоговый режим работы — fail-closed: интерфейс получает статус `1:1 PASS` только тогда, когда все обязательные уровни проверки проходят.

Автор: **Emir Karimov**

## Что проверяет библиотека

В основе проекта лежит каскад из трёх уровней:

| Уровень | Проверка | Что обнаруживает |
|---|---|---|
| A | Структура DOM | отсутствующие или лишние элементы, изменения структуры контролов |
| B | Computed Style | отличия размеров, отступов, цветов, границ и других вычисленных CSS-свойств |
| C | Пиксели | фактические визуальные отличия на изображении |

Итоговое правило:

```text
PASS = A AND B AND C
```

Дополнительно доступны опциональные проверки доступности (`Tier D`) и perceptual/SSIM-сравнение для более устойчивой работы с антиалиасингом.

## Зачем нужен каскад

Один способ сравнения не закрывает все типы регрессий.

Например:

- сравнение классов или общей структуры может не заметить визуально важную потерю дочернего элемента;
- computed styles хорошо показывают конкретное CSS-свойство, но не заменяют проверку структуры;
- пиксельное сравнение подтверждает факт визуального отличия, но не объясняет его причину.

Поэтому `visual-diff` использует независимые проверки и объединяет их в единый fail-closed verdict.

Для снижения ложных структурных отличий предусмотрено сглаживание обёрток с `display: contents`. Кроме того, `sanityCheck()` использует sanity anchors: заведомо одинаковая пара должна проходить, а заведомо сломанная — обнаруживаться.

## Установка

Требуется Node.js 18 или новее.

```bash
npm install @karimover/visual-diff
```

Для браузерного извлечения и Tier C установите Chromium:

```bash
npx playwright install chromium
```

Браузерный модуль использует Chromium через CDP и не требует runtime-зависимости `playwright` в самом пакете. При необходимости путь к конкретному Chromium можно передать через переменную окружения `VISUAL_DIFF_CHROMIUM`.

## Использование

### Сравнение готовых данных без браузера

```js
import {
  diffStructure,
  diffStyle,
  diffPixels,
  cascade,
} from '@karimover/visual-diff';

const A = diffStructure(baseline, candidate);
const B = diffStyle(baseline, candidate);
const C = diffPixels(baselinePng, candidatePng);

const result = cascade(baseline, candidate, {
  baselinePng,
  candidatePng,
});

console.log(result.pass ? '1:1 PASS' : 'BROKEN');
console.log(result.tiers);
```

Нормализованный объект `render` может выглядеть так:

```js
{
  tagHistogram: {
    div: 1,
    span: 2,
    svg: 1,
    path: 1,
  },
  roles: {
    checkbox: 1,
  },
  hasSvg: true,
  control: {
    tag: 'span',
    role: 'checkbox',
    descendantCount: 3,
    tags: {
      span: 1,
      svg: 1,
      path: 1,
    },
  },
  elements: [
    {
      tag: 'span',
      role: 'checkbox',
      styles: {
        width: '16px',
      },
    },
  ],
}
```

### Полный цикл через браузер

```js
import {
  openBrowser,
  loadAndExtract,
  cascade,
  renderReport,
} from '@karimover/visual-diff';
import { writeFileSync } from 'node:fs';

const browser = await openBrowser({
  viewport: { width: 640, height: 320 },
});

try {
  const baseline = await loadAndExtract(browser, baselineHtml, {
    tmpDir: '.tmp',
    name: 'baseline',
  });

  const candidate = await loadAndExtract(browser, candidateHtml, {
    tmpDir: '.tmp',
    name: 'candidate',
  });

  const result = cascade(baseline.struct, candidate.struct, {
    baselinePng: baseline.png,
    candidatePng: candidate.png,
  });

  console.log(
    result.pass ? '1:1 PASS' : 'BROKEN',
    result.tiers,
  );

  writeFileSync(
    'REPORT.html',
    renderReport([
      {
        name: 'my component',
        result,
        baselinePng: baseline.png,
        candidatePng: candidate.png,
      },
    ]),
  );
} finally {
  browser.close();
}
```

`loadAndExtract()` принимает как полный HTML-документ, так и HTML-фрагмент. Для общего CSS можно передать `cssHref`.

## Sanity anchors

Для CI и автоматических пайплайнов полезно проверять саму надёжность gate:

```js
import { sanityCheck } from '@karimover/visual-diff';

const { ok, violations } = sanityCheck([
  {
    name: 'identical',
    expect: 'pass',
    baseline,
    candidate: baselineCopy,
    baselinePng,
    candidatePng,
  },
  {
    name: 'broken',
    expect: 'broken',
    baseline,
    candidate: brokenCandidate,
    baselinePng,
    candidatePng: brokenPng,
  },
]);

if (!ok) {
  throw new Error(
    'gate is untrustworthy: ' + violations.join('; '),
  );
}
```

## CLI

После установки пакет предоставляет команду `visual-diff`.

```bash
npx @karimover/visual-diff baseline.html candidate.html
```

При успешном полном сравнении вывод будет иметь статус `1:1`; при обнаружении регрессии — `BROKEN`.

### Markdown-отчёт

```bash
npx @karimover/visual-diff baseline.html candidate.html --markdown report.md
```

### HTML-отчёт

```bash
npx @karimover/visual-diff baseline.html candidate.html --report report.html
```

### Дополнительные режимы

```bash
npx @karimover/visual-diff baseline.html candidate.html --json
npx @karimover/visual-diff baseline.html candidate.html --a11y
npx @karimover/visual-diff baseline.html candidate.html --perceptual
npx @karimover/visual-diff baseline.html candidate.html --viewport 1280x720
```

Код завершения CLI:

- `0` — `1:1 PASS`;
- `1` — `BROKEN`;
- `2` — ошибка использования или выполнения.

Это позволяет использовать CLI непосредственно в CI/CD.

## Perceptual mode / SSIM

По умолчанию Tier C использует `pixelmatch`.

Для случаев, когда нужно снизить чувствительность к мелкому антиалиасингу, можно включить perceptual mode:

```js
const result = cascade(baseline, candidate, {
  baselinePng,
  candidatePng,
  perceptual: true,
});
```

Также можно передать настройки pixel/perceptual-проверки через `opts.pixels`.

## Accessibility tier

Проверку доступности можно включить отдельно:

```js
const result = cascade(baseline, candidate, {
  a11y: true,
});
```

Tier D может обнаруживать регрессии ARIA и другие различия, которые не обязательно меняют DOM-теги или пиксели.

## Fidelity score

Помимо строгого pass/fail, библиотека предоставляет непрерывную оценку сходства:

```js
import { fidelity } from '@karimover/visual-diff';

const result = fidelity(baseline, candidate);

console.log(result.score);
console.log(result.tiers);
console.log(result.explain);
```

`fidelity()` возвращает значение от `0` до `1` и предназначен как дополнительная метрика близости. Он не заменяет строгий `cascade()` и не должен использоваться как единственный gate.

## Генерация отчётов и badges

HTML-отчёт:

```js
import { renderReport } from '@karimover/visual-diff';
```

Markdown для PR-комментариев:

```js
import { renderMarkdown } from '@karimover/visual-diff';
```

Badges для shields.io:

```js
import {
  badgeFromFidelity,
  badgeFromVerify,
} from '@karimover/visual-diff';
```

Обе функции возвращают JSON в формате endpoint badge.

## Verify API

`verify()` предоставляет удобный CI/agent-friendly интерфейс:

```js
import { verify } from '@karimover/visual-diff';

const result = verify(before, after, options);

console.log(result.verdict);
console.log(result.pass);
console.log(result.tiers);
console.log(result.why);
```

Формат результата:

```js
{
  verdict: '1:1' | 'BROKEN',
  pass: true,
  tiers: {
    A: true,
    B: true,
    C: true,
  },
  why: null,
}
```

При ошибке `why` объясняет, какой уровень проверки не пройден.

## Self-proof

Проект содержит собственную end-to-end проверку:

```bash
npm run proof
```

Скрипт создаёт две пары UI:

1. идентичную — ожидается `PASS`;
2. намеренно сломанную — ожидается `BROKEN`.

Результат сохраняется в:

```text
examples/out/REPORT.html
```

Такой подход дополнительно проверяет, что gate не пропускает известную регрессию и не отклоняет корректную пару.

## Тесты

```bash
npm test
```

В проекте используется встроенный Node.js test runner.

Тесты покрывают структуру, computed styles, пиксельное сравнение, cascade, perceptual mode, accessibility, temporal checks, receipts, Markdown/HTML reports, badges, fidelity и producer-agnostic contract.

## CI

Workflow GitHub Actions находится в:

```text
.github/workflows/ci.yml
```

CI выполняет:

```text
npm ci
npm audit --audit-level=high
npm test
npx playwright install --with-deps chromium
npm run proof
```

Также сохраняется HTML-отчёт self-proof как CI artifact.

## API

Основные экспортируемые функции:

- `diffStructure(baseline, candidate)` — Tier A;
- `diffStyle(baseline, candidate, { tol, props })` — Tier B;
- `diffPixels(baselinePng, candidatePng, { threshold, includeAA, flagPct })` — Tier C;
- `cascade(baseline, candidate, options)` — объединённый verdict;
- `sanityCheck(anchors, options)` — проверка надёжности gate;
- `verify(before, after, options)` — CI-friendly verdict;
- `fidelity(baseline, candidate, { style })` — числовая оценка сходства;
- `openBrowser(options)` — запуск браузера через CDP;
- `loadAndExtract(browser, html, options)` — получение нормализованного render + PNG;
- `renderReport(cards, options)` — HTML-отчёт;
- `renderMarkdown(cards, options)` — Markdown-отчёт;
- `badgeFromFidelity(result, options)` — badge JSON для fidelity;
- `badgeFromVerify(result, options)` — badge JSON для verify.

В библиотеке также экспортируются вспомогательные функции для diff и работы с браузерным извлечением.

## Требования

- Node.js `>=18`;
- для browser extractor и self-proof — доступный Chromium;
- для обычных pure-tier проверок браузер не требуется.

## Структура проекта

```text
src/                    исходный код библиотеки
bin/                    CLI
examples/               пример end-to-end использования и self-proof
experiments/            экспериментальные модули, на основе которых развивались проверки
loops/                  заметки и результаты циклов разработки
test/                   unit и contract tests
.github/workflows/      CI
```

## Лицензия

MIT License.

Copyright (c) 2026 Emir Karimov
