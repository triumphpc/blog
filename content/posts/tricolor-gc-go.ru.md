---
title: "Трёхцветный GC в Go: как маркер не теряет живые объекты — и что будет, если убрать барьер записи"
date: 2026-09-12
draft: false
tags: ["go", "golang", "gc", "runtime", "memory", "write-barrier", "internals"]
categories: ["go", "engineering"]
summary: "Трёхцветная маркировка — это не «алгоритм сборки мусора», а протокол сосуществования коллектора и программы, которая прямо во время обхода переписывает граф объектов. Разбираю инвариант, барьер записи (включая гибридный барьер Go 1.8), рождение объектов чёрными и mark assist — и даю интерактивную песочницу, где можно выключить барьер и своими глазами увидеть, как GC освобождает живой объект."
ShowToc: true
series: ["Go Internals"]
---

Трёхцветная маркировка обычно объясняется так: белое — не найдено, серое — в очереди, чёрное — просканировано. Всё верно и ровно настолько же бесполезно, потому что главный вопрос остаётся за кадром: зачем вообще нужны три цвета, если для обхода графа достаточно двух — «посещён» и «не посещён»?

Так вот. Третий цвет существует ровно по одной причине: программа не останавливается на время сборки. И этот пост — про то, что из этого следует. В конце — интерактивная песочница, где барьер записи можно выключить и посмотреть, как GC съедает живой объект.

## Зачем нужен третий цвет

Наивный сборщик делает mark & sweep за одну <abbr title="Stop-the-world — пауза, на время которой рантайм останавливает все горутины">STW</abbr>-паузу: остановили мир, обошли граф от корней, освободили всё, до чего не дошли. Корректно, просто — и пауза линейна по размеру живой кучи. На 10 ГБ живых данных это секунды, а не микросекунды.

Go пошёл другим путём ещё в 1.5: маркировка идёт **конкурентно** с программой ([Go 1.5 GC: prioritizing low latency and simplicity](https://go.dev/blog/go15gc)). Мир останавливается дважды и ненадолго — на включение барьера со сканированием корней и на завершение маркировки. С Go 1.8 эти паузы «обычно меньше 100 микросекунд, а часто — порядка 10» ([Go 1.8 release notes](https://go.dev/doc/go1.8#gc)), и что важно — они больше не зависят от размера кучи.

Но конкурентность создаёт проблему, которой в STW-сборщике не существует. Пока коллектор обходит граф, **мутатор** (термин из [Dijkstra et al., 1978](https://dl.acm.org/doi/10.1145/359642.359655) — программа, которая меняет граф объектов прямо во время обхода) переписывает указатели. И вот тут двух цветов уже не хватает.

Представьте: коллектор полностью просканировал объект `B` — прочитал все его поля, больше к нему не вернётся. В этот момент горутина выполняет `B.next = W`, где `W` — ещё не найденный белый объект, и одновременно обнуляет единственную другую ссылку на `W`. Формально `W` жив: он достижим от корней через `B`. Фактически коллектор его никогда не найдёт — в `B` он уже был.

Это классическая потеря объекта. Три цвета нужны, чтобы её поймать: «просканирован» (чёрный) и «найден, но не просканирован» (серый) — принципиально разные состояния, и правило корректности формулируется именно через них.

<figure class="svg-diagram">
<svg viewBox="0 0 760 250" role="img" aria-label="Три стадии потери живого объекта: маркер проходит B, мутатор перевешивает указатель на W за чёрный объект, очистка освобождает живой W">
  <defs>
    <marker id="ru-ar" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M0 0L10 5L0 10z" fill="#8b90a0"/>
    </marker>
    <marker id="ru-ar-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M0 0L10 5L0 10z" fill="#fbbf24"/>
    </marker>
  </defs>

  <!-- panel 1 -->
  <rect x="8" y="34" width="236" height="200" rx="8" fill="none" stroke="#262a31"/>
  <text x="20" y="24" fill="#8b90a0" font-size="12" font-family="var(--font-mono,'JetBrains Mono'),monospace">1 · маркер прошёл B</text>
  <circle cx="70" cy="88" r="22" fill="#e5e7eb" stroke="#8b90a0" stroke-width="1.5"/>
  <text x="70" y="93" fill="#0d0e12" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">B</text>
  <text x="70" y="128" fill="#8b90a0" font-size="10.5" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">чёрный</text>
  <circle cx="70" cy="180" r="22" fill="none" stroke="#8b90a0" stroke-width="1.5"/>
  <text x="70" y="185" fill="#e5e7eb" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">X</text>
  <circle cx="188" cy="180" r="22" fill="none" stroke="#e5e7eb" stroke-width="1.5" stroke-dasharray="4 3"/>
  <text x="188" y="185" fill="#e5e7eb" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">W</text>
  <text x="188" y="220" fill="#8b90a0" font-size="10.5" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">белый</text>
  <line x1="94" y1="180" x2="162" y2="180" stroke="#8b90a0" stroke-width="1.6" marker-end="url(#ru-ar)"/>

  <!-- panel 2 -->
  <rect x="262" y="34" width="236" height="200" rx="8" fill="none" stroke="#fbbf24" stroke-opacity="0.5"/>
  <text x="274" y="24" fill="#fbbf24" font-size="12" font-family="var(--font-mono,'JetBrains Mono'),monospace">2 · мутатор: *B = W; X = nil</text>
  <circle cx="324" cy="88" r="22" fill="#e5e7eb" stroke="#8b90a0" stroke-width="1.5"/>
  <text x="324" y="93" fill="#0d0e12" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">B</text>
  <circle cx="324" cy="180" r="22" fill="none" stroke="#8b90a0" stroke-width="1.5" stroke-opacity="0.35"/>
  <text x="324" y="185" fill="#8b90a0" font-size="14" font-weight="700" text-anchor="middle" fill-opacity="0.45" font-family="var(--font-mono,'JetBrains Mono'),monospace">X</text>
  <circle cx="442" cy="180" r="22" fill="none" stroke="#e5e7eb" stroke-width="1.5" stroke-dasharray="4 3"/>
  <text x="442" y="185" fill="#e5e7eb" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">W</text>
  <line x1="336" y1="108" x2="430" y2="158" stroke="#fbbf24" stroke-width="1.8" marker-end="url(#ru-ar-a)"/>
  <line x1="348" y1="180" x2="416" y2="180" stroke="#8b90a0" stroke-width="1.4" stroke-opacity="0.3" stroke-dasharray="3 4"/>
  <text x="380" y="136" fill="#fbbf24" font-size="10.5" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">белый за чёрным</text>

  <!-- panel 3 -->
  <rect x="516" y="34" width="236" height="200" rx="8" fill="none" stroke="#f87171" stroke-opacity="0.5"/>
  <text x="528" y="24" fill="#f87171" font-size="12" font-family="var(--font-mono,'JetBrains Mono'),monospace">3 · sweep</text>
  <circle cx="578" cy="88" r="22" fill="#e5e7eb" stroke="#8b90a0" stroke-width="1.5"/>
  <text x="578" y="93" fill="#0d0e12" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">B</text>
  <circle cx="696" cy="180" r="22" fill="none" stroke="#f87171" stroke-width="1.6" stroke-dasharray="5 4"/>
  <text x="696" y="185" fill="#f87171" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">W</text>
  <text x="696" y="220" fill="#f87171" font-size="10.5" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">освобождён · жив</text>
  <line x1="590" y1="108" x2="684" y2="158" stroke="#f87171" stroke-width="1.6" stroke-opacity="0.5" stroke-dasharray="4 4"/>
</svg>
<figcaption>Маркер не возвращается в чёрные объекты — значит, указатель, появившийся в чёрном объекте уже после сканирования, нужно ловить отдельно</figcaption>
</figure>

## Инвариант и барьер

Правило, которое держит всю конструкцию, формулируется в двух вариантах.

**Сильный инвариант**: чёрный объект никогда не содержит указатель на белый. **Слабый инвариант**: чёрный объект может указывать на белый, но только если этот белый достижим ещё и по цепочке от какого-нибудь серого. Слабый вариант допускает больше состояний и потому дешевле в поддержке — Go опирается именно на него.

Соблюдать инвариант нужно в момент, когда его можно нарушить, то есть при записи указателя в кучу. Отсюда **барьер записи** (write barrier) — небольшой фрагмент кода, который компилятор вставляет перед каждой такой записью. В наивном варианте (Dijkstra) он делает ровно одно: если записываемое значение белое — красит его в серое, то есть кладёт в очередь маркера. Белый объект, спрятанный за чёрным, немедленно перестаёт быть белым.

Реальный Go с версии 1.8 использует **гибридный барьер** — комбинацию delete-барьера Yuasa и insert-барьера Dijkstra ([proposal 17503, Clements & Hudson](https://github.com/golang/proposal/blob/master/design/17503-eliminate-rescan.md)). Псевдокод прямо из комментария в [`runtime/mbarrier.go`](https://github.com/golang/go/blob/master/src/runtime/mbarrier.go):

```go
writePointer(slot, ptr):
    shade(*slot)
    if current stack is grey:
        shade(ptr)
    *slot = ptr
```

Первая строка — это и есть отличие от учебного варианта: красится **старое** значение слота, а не только новое. Смысл в том, что коллектор получает snapshot графа на момент начала цикла: указатель, который мутатор разрывает, всё равно попадёт в очередь. Именно это позволило Go убрать финальное пересканирование стеков под STW — самую неприятную паузу в схеме до 1.8, которая линейно росла с числом горутин.

Барьер стоит денег на каждой записи указателя, и поэтому в Go его включают только на время цикла — вне маркировки он выключен и код идёт по быстрому пути.

Дальше — два следствия, которые в учебниках обычно упоминают вскользь, а на практике они и делают схему рабочей.

**Объекты, выделенные во время цикла, рождаются чёрными.** Не серыми — чёрными. Логика простая: объект только что создан, внутри него ещё нечего сканировать, а если красить его в серый, маркер будет бесконечно догонять аллокации активной горутины. Цена — такие объекты гарантированно доживают до следующего цикла, даже если мусор. Это осознанный размен: floating garbage в обмен на завершаемость.

**Mark assist.** Горутина, которая аллоцирует, обязана сама поработать маркером пропорционально выделенному объёму. Без этого одна аллоцирующая горутина легко опережает фоновых маркеров, куча растёт быстрее, чем сканируется, и цикл не сходится. Ассист — это встроенный в аллокатор механизм обратного давления; детали пропорций живут в [pacer](https://github.com/golang/proposal/blob/master/design/44167-gc-pacer-redesign.md), переписанном в Go 1.18.

Собственно, поэтому «GC съедает CPU» на практике часто выглядит как «аллоцирующие горутины тормозят»: время уходит не в фоновых воркерах, а в ассистах.

<figure class="svg-diagram">
<svg viewBox="0 0 760 130" role="img" aria-label="Фазы цикла GC в Go: короткий STW, конкурентная маркировка, короткий STW, конкурентная очистка">
  <rect x="8" y="34" width="96" height="44" rx="6" fill="#f87171" fill-opacity="0.16" stroke="#f87171"/>
  <text x="56" y="60" fill="#e5e7eb" font-size="11.5" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">STW</text>
  <text x="56" y="96" fill="#8b90a0" font-size="10" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">барьер on,</text>
  <text x="56" y="110" fill="#8b90a0" font-size="10" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">корни</text>

  <rect x="112" y="34" width="308" height="44" rx="6" fill="#4ade80" fill-opacity="0.14" stroke="#4ade80"/>
  <text x="266" y="60" fill="#e5e7eb" font-size="11.5" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">конкурентная маркировка</text>
  <text x="266" y="96" fill="#8b90a0" font-size="10" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">серое → чёрное · программа работает · барьер ловит записи · mark assist</text>

  <rect x="428" y="34" width="84" height="44" rx="6" fill="#f87171" fill-opacity="0.16" stroke="#f87171"/>
  <text x="470" y="60" fill="#e5e7eb" font-size="11.5" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">STW</text>
  <text x="470" y="96" fill="#8b90a0" font-size="10" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">финиш</text>

  <rect x="520" y="34" width="232" height="44" rx="6" fill="#60a5fa" fill-opacity="0.14" stroke="#60a5fa"/>
  <text x="636" y="60" fill="#e5e7eb" font-size="11.5" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">конкурентная очистка</text>
  <text x="636" y="96" fill="#8b90a0" font-size="10" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">всё, что осталось белым, — мусор</text>

  <text x="8" y="22" fill="#8b90a0" font-size="10.5" font-family="var(--font-mono,'JetBrains Mono'),monospace">ширина блоков условна: STW-паузы — десятки микросекунд, маркировка — миллисекунды и больше</text>
</svg>
<figcaption>Один цикл GC в Go: мир останавливается дважды и ненадолго, вся тяжёлая работа идёт параллельно с программой</figcaption>
</figure>

Ещё одно свойство, которое стоит проговорить: трёхцветная маркировка спокойно собирает **циклы мусора**. `A → B → A`, недостижимые от корней, останутся белыми и будут освобождены — никакого специального кода для этого не нужно. Это фундаментальное преимущество перед подсчётом ссылок, где циклическая связка живёт вечно.

## Песочница

Дальше — интерактивная модель одного цикла маркировки. Слева root set: стеки горутин и глобалы. Справа куча: объекты со связями, которые можно перетаскивать, связывать и рвать.

{{< sandbox src="sandbox/tricolor-gc.ru.html" height="900" title="Трёхцветный GC в Go — интерактивная песочница" link="Открыть песочницу в отдельной вкладке" caption="Кликните по двум объектам, чтобы связать их; по стрелке — чтобы удалить указатель" >}}

Что стоит попробовать по порядку:

1. **«Шаг цикла»** — и смотрите на панель «Серый набор». Это очередь маркера. Каждый шаг: достали объект из очереди, прочитали его указатели, белые цели → серые, сам объект → чёрный. Цикл заканчивается, когда очередь пуста.
2. **Включите мутатор.** Теперь параллельно с маркировкой идут записи указателей и аллокации. Обратите внимание, что новые объекты появляются сразу чёрными — то самое allocate-black, — и в журнале видно, как срабатывает барьер.
3. **Выключите барьер записи и нажмите «Эксперимент: потеря объекта».** Мутатор перевешивает белый объект за чёрный и обнуляет исходную ссылку. Маркер в чёрное не возвращается — на фазе очистки живой объект уходит в освобождённые. Ровно тот баг, который барьер предотвращает.
4. Включите барьер обратно и повторите эксперимент. В момент записи белый станет серым — в журнале появится строка `[барьер]`, объект доживёт до конца цикла.
5. Досмотрите первый цикл до конца и найдите пару `F ⇄ G` — это мусорный цикл, который освобождается без всякого специального кода.

Одна честная оговорка по модели: барьер в песочнице — это классический insert-барьер Dijkstra, «чёрный получил указатель на белый → красим белый в серый». Настоящий Go, как я писал выше, использует гибридный барьер и красит ещё и старое значение слота. Для того чтобы увидеть, зачем барьер вообще нужен, insert-варианта достаточно; для чтения `mbarrier.go` — уже нет.

## Что изменилось с Green Tea

Всё вышесказанное — про инвариант и барьер — по-прежнему верно, но сам обход кучи в свежих версиях Go устроен иначе. В Go 1.25 появился экспериментальный сборщик Green Tea ([issue #73581](https://github.com/golang/go/issues/73581)), а в Go 1.26 он [включён по умолчанию](https://go.dev/doc/go1.26); старая схема осталась за флагом `GOEXPERIMENT=nogreenteagc`, и её планируют убрать в 1.27.

Идея в том, что классический маркер ходит по отдельным указателям, разбросанным по памяти, и на небольших объектах это убивает локальность кэша: каждый шаг очереди — почти гарантированный промах. Green Tea работает не с отдельными объектами, а со **страницами**: серым становится не объект, а спан, и маркер обрабатывает его целиком, последовательно по памяти. Заявленный выигрыш — сокращение накладных расходов GC на 10–40% на реальных программах, плюс ещё около 10% на свежих amd64 (Ice Lake / Zen 4 и новее) за счёт векторных инструкций по битмапам страницы.

Что важно для этой статьи: поменялась **единица работы маркера**, а не протокол. Три цвета, инвариант и барьер записи никуда не делись — это по-прежнему трёхцветная конкурентная маркировка, просто очередь теперь хранит страницы, а не объекты.

## Ссылки

- [Dijkstra, Lamport, Martin, Scholten, Steffens. On-the-Fly Garbage Collection: An Exercise in Cooperation, CACM 21(11), 1978](https://dl.acm.org/doi/10.1145/359642.359655) — первоисточник трёхцветной абстракции
- [Go proposal 17503: Eliminate STW stack re-scanning](https://github.com/golang/proposal/blob/master/design/17503-eliminate-rescan.md) — гибридный барьер, Go 1.8
- [`runtime/mbarrier.go`](https://github.com/golang/go/blob/master/src/runtime/mbarrier.go) — комментарии в этом файле лучше большинства статей про GC
- [A Guide to the Go Garbage Collector](https://go.dev/doc/gc-guide) — официальный гайд: GOGC, GOMEMLIMIT, latency
- [Go 1.26 release notes](https://go.dev/doc/go1.26) — Green Tea по умолчанию
