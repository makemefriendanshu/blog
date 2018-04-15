---
title_image_path: parallel.jpg
category: Програма
created_at: 2017-04-19T21:16:17
tags:
  - elixir
  - erlang
  - process
  - spawn
  - receive
  - send
  - schedulers
  - erlang history
  - concurrency
  - parallelism
  - scheduling
---

# Процеси

В тази статия ще си говорим за основните градивни единици на всяка програма в `Elixir` - процесите.
Досега ги споменавахме на доста места, защото нямаше как да избегнем това. Всъщност процесите в `Elixir`
идват от `Erlang`. Когато включим и процесите в описанието на езика, за `Elixir` можем да кажем следното:

`Elixir` е език изграден от няколко 'слоя', като всеки от тях е надграждане на по-долните слоеве.
* Най-малкият слой е функционален `Elixir` - функционалният език с който се занимавахме досега. Доста стандартен функционален език с някои приятни идеи като съпоставянето на образци и `pipe` оператора.
* Следващият слой е конкурентен `Elixir` - Добавяйки процесите и съобщенията, които си обменят, получаваме точно това. Във всеки процес върви функционален код. Самите процеси не са функционални структури.
* Ако добавим и отдалечена комуникация между процеси, получаваме дистрибутиран `Elixir`.
* Последният слой е съставен от библиотеките на `OTP` и създаването на връзки и монитори между процеси - `fault-tolerant Elixir`.

В тази статия ще си говорим за конкурентния `Elixir`.

## Малко история

За да разберем защо моделът за конкурентност на `Elixir` е такъв какъвто е, трябва да погледнем назад.
Трябва да погледнем в `Erlang` и причините на създаването му.

Както може би знаете, `Erlang` е създаден в лаборатория на `Ericsson` през 80-те години.
Основната му идея е да е способ за писане на конкурентни програми, които трябва да могат да се изпълняват безкрайно.

Всъщност изискването поставено на _Joe Armstrong_ е да измисли "по-добър начин за писане на телекомуникационни програми".
Още 80-те години, този тип програми са имали изискването да бъдат конкурентни (една програма трябва да може да поддържа хиляди едновременни транзакции).
Освен това, задължително е трябвало да бъдат толерантни към грешки и проблеми, както софтуерни, така и хардуерни.
Както и да имат практически нулев `downtime`.
С други думи да са винаги работещи, кодът им да може да се заменя с по-нови версии,
докато те работят.

Както виждате тогавашните проблеми при телекомуникационните програми са всъщност днешните проблеми на интернет сървърите със стотици хиляди потребители.
Именно затова езици като `Elixir` и `Erlang` набират популярност днес.

В средата на 80-те `Ericsson` са използвали устройства наречени `AXE`, за осъществяване на връзка между потребители.
Те са се програмирали на език, наречен `PLEX`, който е работил за горните изисквания, но е бил много, много труден за употреба
и освен това тясно свързан с хардуера на това устройство - `AXE`. Идеята на `Erlang` е била да бъде нещо като `PLEX`, но да може
да върви на различни типове хардуер, както и да е по-бърз и лесен за писане.

В началото `Erlang` е много повлиян от `PLEX`.
В `PLEX` информацията се е копирала от компонент на компонент за да се избегнат грешки със споделен достъп до ресурси.
Софтуерът е следвал следната спецификация:
* Множество паралелни процеси живеят в паметта.
* По всяко време, повечето от тях чакат събитие, което може да е провокирано или от съобщение пратено към тях, или от _timer_.
* Когато даденото събитие се случи, процесът прави някакво малко изчисление, променя някак състоянието си или изпраща съобщение до други процеси. След това пак започва да чака за ново събитие.

Тези процеси трябва да са много леки и лесни за създаване, което е отново наследено от `AXE/PLEX` системите.
Това значи, че процесите трябва да са част от самия език, а не от операционната система, на която той се изпълнява.

Следващото нещо, наследено от `AXE/PLEX` е, че при грешка само текущата транзакция ще бъде невалидна.
Грешките в един процес не могат да влияят на другите процеси. Софтуерът трябва да продължи да работи.

По онова време паралелизъм означава множество устройства, които работят с даденият софтуер и се възприемат като едно.
Тоест езикът трябва да е лесен за дистрибутивност. И така получихме свойствата на съвременните `Erlang/Elixir`.

За да са валидни тези свойства. Езикът е разработен така:
1. Кодът върви в процеси, които са на ниво език. Подобни на `green thread`-ове в други езици.
2. Тези процеси не споделят памет - имат собствен стек и собствен `heap`.
3. Много са лесни за създаване и си комуникират чрез размяна на съобщения.
4. Лесно могат да си комуникират помежду си, дори да са на различни машини (това се случва малко по-късно в живота на езика).
5. Ако един процес 'умре', другите продължават да живеят. Може нов да го замести, зависи от стратегията.

Този модел е различен от моделът, популярен навремето, а и сега, в който има споделена памет и нишки работят с
различни ресурси, заключвайки и отключвайки ги за достъп.

Езикът `Erlang` започва като библиотека на `Prolog` за `fault-tolerant and distributed` програмиране, но се развива бързо
като диалект на пролог и после като самостоятелен език. Първият интерпретатор на езика е на `Prolog`. `Erlang` е повлиян донякъде и от `Smalltalk`.
Първата имплементация на _Joe_ за изпращане на 'звънящо' съобщение до телефон е на `Smalltalk`. Влиянието на `Smalltalk` може
да се види в размяната на съобщения между процесите. Между другото в `Simula`, един от първите обектно-ориентирани езици, извикването
на метод на обект се нарича, изпращане на съобщения. По същия начин това повлиява за създаването на `Actor` модела за конкурентност.

Правилно е да се каже, че `Erlang` не е повлиян от и не имплементира `Actor` модела. Но процесите на `Erlang` и актьорите имат общ
предшественик - комуникацията между обекти със съобщения. Доста от идеите за Актьорите намират своят път в процесите на `Erlang` независимо от `Actor` модела.
Често грешно наричат `Erlang` език, имплементиращ `Actor` модела. Все пак може да се каже, че процесите са актьори (държат се като актьори), чиито вътрешности са функционални.
Във всеки процес върви функционален език. Така че вътрешността на един процес няма нищо общо с `Actor` модела.

И така `Erlang` се превръща от `Prolog`, който поддържа конкурентност в обособен език, по който започва да работи още един човек - _Robert Virding_.
Двамата с _Joe Armstrong_ оформят паралелно два интерпретатора на `Erlang`, написани на `Prolog`.
Това все още остава прототип в лаборатория, но придобива потребители - 3-ма човека. Това че ги има тези хора, тестващи и използващи езика, подпомага
много за развитието му. Да речем процесите започват да имат специален буфер, наречен 'кутия за съобщения' `Mailbox`, започват да могат да създават връзки помежду си.
Така ако някой от тях получи грешка, друг може да бъде уведомен със специално съобщение и да реагира.

В края на 1989 година, езикът е тестван и функционалността му е намерена за задоволителна.
Проблемът е че е много бавен.
Излиза ново изискване - да го направят поне `40` пъти по-бърз, което после се увеличава.
Така се ражда първата абстрактна машина на `Erlang`, написана на `C` - `JAM` ( _Joe's Abstract Machine_ ).
Преди `C` и други езици са били разглеждани, и други абстрактни машини са били разучавани.
Третият създател на `Erlang`, _Mike Williams_ е с доста повече опит от _Joe_ в `C`, затова той написва `JAM`.

И така 90-те години бележат началото на изхвърляне на доста `Prolog` синтаксис от `Erlang`, по-добри `GC` стратегии, `binary` информация над даден размер да се пази в общ `heap` за даден `node` и други.

През 1993 _Bogumil (Bogdan) Hausman_ създава `TEAM`, прекръстена после на `BEAM` - доста по-оптимизирана машина за изпълнение на `Erlang bytecode`.

Две хубави неща се случват за `Erlang`:
1. В края на 1995, проекта за `AXE-N` устройствата се сгромолясва. Те са щели да работят с друг, специфичен за тях език, а не с `Erlang`. Това води до широкото използване на устройства, програмирани на `Erlang`.
2. През 1998 година _Ericsson Radio AB_ забранява `Erlang` за ползване. Решението идва с идеята, фирмата и нейните продукти да не зависят от кръга хора около `Erlang`. Това обаче кара тези хора да напуснат `Ericsson` и довежда до отварянето на `Erlang`.

Първото събитие, сгромолясването на `AXE-N`, води до широкото използване на `Erlang` в `Ericsson`.
Това пък е причина за разработката на `framework`-а `OTP`. От там нататък `Erlang` и `OTP` се разпространяват заедно.
Библиотеката `OTP` съдържа:
* Множество малки помощни библиотеки на `Erlang`.
* `Design Patterns` за програмирани на често желани програми.
* Документация, курсове и `How to`-та
* `Mnesia`/`ETS` бази данни

Отварянето на кода на `Erlang`, пък, води до неговото популяризиране и по-масово използване.

С развитието на `IOT` и `instant messaging` програмите, както и програми, обслужващи хиляди `request`-и на секунда, моделът на `Erlang` става все по-актуален.
От 2006 `BEAM` започва да поддържа паралелно изпълнение на процесори с множество ядра. А както знаете, преди няколко години `Elixir` се ражда върху `BEAM`.

Та всичко се върти около тези `Erlang` процеси.
Езикът е измислен около тях и комуникацията между тях.
_Joe Armstrong_ нарича `Erlang` език за конкурентно-ориентирано програмиране, като се базира на няколко правила какво означава това.

Правилата са:
* Системата е изградена от процеси.
* Процесите не споделят нищо.
* Процесите си комуникират чрез асинхронно изпращане на съобщения.
* Процесите са изолирани един от друг.

В заключение:
1. 1986 : `Erlang` е декларативен език с добавена способност за конкурентно изпълнение.
2. 1995 : `Erlang` е функционален език с добавена способност за конкурентно изпълнение.
3. 2005 : `Erlang` е конкурентно-ориентиран език, който се състои от комуникиращи си компоненти, написани на функционален език.

Всички това се върти около тези компоненти - процесите. И тъй като `Elixir` е роден върху `BEAM`, можем да кажем същото за него.
Нека сега видим как да създаваме и работим с процеси.

## Създаване на процеси

Един от начините да създадем нов процес в `Elixir` е чрез `spawn/1`:

```elixir
spawn(fn ->
  <изрази>
end)
```

Нека направим следната функция:

```elixir
execute_after_action = fn (action, milliseconds) ->
  :timer.sleep(milliseconds)
  result = action.()
  IO.puts(result)
end
```

Идеята е изкуствено да имаме действие, което отнема дадено време да се изпълни.

```elixir
execute_after_action.(fn -> "Awake!" end, 1000)
# След една секунда ще видим Awake! Дотогава програмата чака.
```

Сега можем да пуснем това действие да се изпълни в нов процес:

```elixir
spawn(fn -> execute_after_action.(fn -> "Awake!" end, 1000) end)
IO.puts("Sleeping...")
```

Това което ще се случи е, че веднага след като пуснем процеса ще видим на екрана
текста _Sleeping..._, а след една секунда ще видим _Awake_.

Друга форма на `spawn` е `spawn/3`. Тази функция има три аргумента, често наричани `MFA`.
* `M` - означава модул, и е точно това - модул.
* `F` - означава функция и трябва да бъде атом, представляващ името на публична функция от модула `M`.
* `A` - това са аргументите, които трябва да се предадат на тази функция. Те са във формата на списък.

```elixir
defmodule Executor do
  def action_after(action, milliseconds) do
    :timer.sleep(milliseconds)
    IO.puts(action.())
  end
end

spawn(Executor, :action_after, [fn -> "Finally!" end, 1000])
# Текущият процес няма да блокира.
# След една секунда ще видим 'Finally!' на екрана.
```

И така, `spawn` е една от функциите за създаване на процеси. Създадените нови процеси
се изпълняват конкурентно на текущия процес. Казваме 'текущия', защото и кодът който написахме
преди малко се изпълнява в процес. Пример е `iex`. Интерпретаторът `iex` също се изпълнява в процес.
Новите процеси се изпълняват конкурентно, а е възможно и да се изпълняват паралелно.

## Конкурентност и паралелизъм

Няколко пъти казахме, че `Erlang/Elixir` е конкурентна платформа. Нека да уточним какво означава това.

Да си представим опашка в магазин - ако магазинерката е само една, всички чакат и си маркират продуктите един след друг.
Тук няма никаква конкурентност, всяка транзакция на продукти се случва след приключване на предходната.
Ако обаче отвори втора каса, опашките стават две, маркирането на продукти става по конкурентен начин. Две транзакции могат да вървят по едно и също време.
В този пример имаме и паралелизъм. Но конкурентност и паралелизъм не са едно и също нещо.

Друг пример - нека имаме автор на статия и редактор. Двамата работят конкурентно върху статията; Авторът пише един абзац, редакторът минава над този абзац и го редактира,
през това време авторът чака, след това пише втори абзац, редакторът чака, след това редакторът редактира. Така имаме конкурентност - и двамата работят по статията конкурентно.
Можеше да нямаме - авторът първо да напише всичко, а редакторът да редактира след това. Сега паралелно би било следното. Авторът да напише първия абзац, да го прати на редактора
и веднага да почне да работи по втория абзац, докато по същото време редакторът редактира първия.

Ако имаме много конкурентни парчета код които се изпълняват в няколко процеса сме конкурентни, ако обаче имаме няколко машини или процесорни ядра, на които се изпълняват сме и паралелни.
Да речем `AXE` устройствата, за които говорихме, са поддържали паралелизъм, защото са били съставени от множество малки `switch` устройства, на всяко от които са вървели процеси.

## Паралелизъм в `Erlang/Elixir`

Когато стартираме `Elixir`, той върви в един `OS process` или една `BEAM` инстанция, която наричаме `node`.
За всяко ядро на `CPU`-то си, обикновено получаваме по една `OS-level` нишка.
Във всяка такава нишка се изпълнява нещото, наречено `Scheduler`.
Тези `Scheduler`-и обикновено са обвързани с ядро на процесора, но е възможно и да ги сменят.
Има си флагове, когато стартираме `Elixir` за определяне на поведението им.

Какво прави един такъв `Scheduler`? Управлява опашка, наречена `run queue`.
Като цяло това е приоритетна опашка от `Elixir` процеси и портове. Това значи, че ако имаме четири
`Scheduler`-а, е възможно да имаме четири паралелни `Elixir-level` процеса, защото всеки от
тях е на различно ядро и управлява различна опашка от процеси.

Какво става като извикаме `spawn`? Нов процес се създава и се поставя в някоя от опашките на `Scheduler`-ите.
Обикновено един такъв `Elixir-level` процес е голям около 1KB-2KB при създаването си (за разлика от OS-level нишките които обикновено взимат няколко мега байта само за стека си).
Това означава, че можем да създаваме огромен брой процеси без да се притесняваме. Говорим за милиони.

Интересно нещо е миграцията между `Scheduler`-и. Възможно е процес да смени опашката си и да започне да се управлява от друг `Scheduler`.
Има сложен алгоритъм за балансиране на натоварването между ядрата, който е отговорен за тази миграция.
Често ако `Scheduler` остане без работа може да си 'поиска' процеси от други опашки.
Между другото, ако няма достатъчно работа, някои от `Scheduler`-ите няма въобще да бъдат стартирани, докато не се наложи.
Стартирането и спирането на `Scheduler`-и е скъпа операция, така че се извършва сравнително рядко и с отлагане.

![alt text](/custom/assets/schedulers_now.png "Scheduler-и и опашки") {: .inline-image}

Добре е да се спомене, че `Sheduler`-ите освен `Elixir` процеси управляват и портове.
Портовете са начинът по който `Elixir` комуникира с външния свят. Да речем с такива портове
правим комуникация с други `OS-level` процеси, написани на друг език.

Процесите и портовете имат право на до `N` редукции. В текущата версия на `BEAM` това `N` е `2000`.
Всяка операция свързана с процес е редукция. Да речем извикването на функция или макрос, изпращане на съобщение,
`GC` в `heap`-a на дадения процес и така нататък. Когато текущо-изпълняващ се процес изчерпа редукциите си или
пък е в очакване на нещо и не прави нищо, той става неактивен и `Scheduler`-а активизира друг от опашката.
Това се отнася и за портовете - `IO` операции, комуникация с други `OS-level` процеси, изпращане на съобщения между
дистрибутирани `BEAM` инстанции - всичко струва редукции. По тежките операции са по скъпи.

Тази стратегия на планиране се нарича превантивна ( _preemptive_ ). В общи линии `Scheduler`-ът решава
кога да прекъсне процес, използвайки броя на редукциите, времето или приоритета на процеса.
За разлика от повечето езици, които използват кооперативна стратегия, в `Elixir`, `Scheduler`-ът може
да прекрати изпълнението на задача 'насила'. По този начин няма как много тежък откъм операции процес да е
активен много дълго време, блокирайки всички други в опашката си.

Нека сега да видим как си взаимодействат процесите.

## Комуникация между процеси

Има три основни функции за работа с процеси:
* `spawn` ги създава. Запознахме се с нея.
* `send` изпраща съобщение до процес.
* `receive` чака за съобщения към текущия процес.

Нека пак да разгледаме `spawn`.
Вече знаем с какви аргументи работи и какво прави - създава нов процес.
Сега нека поговорим за това, което връща. Връща инстанция на типа `PID`.
Всъщност `PID`-ът представлява адреса на процеса.
Може да бъде използван за изпращане на съобщения.

Както знаете процесите са напълно изолирани един от друг.
Начинът по който обменят информация е чрез тези съобщения, които копират данни от `heap`-а на един процес към друг.
Функцията `send/2` приема `PID` на процеса, към който искаме да изпратим съобщение и самото съобщение:

```elixir
pid = spawn(action)

send(pid, message)
```

Третата важна функция при процесите, `recieve`, прилича малко на `case`.
В общи линии всеки процес си има опашка от съобщения.
Когато дадено съобщение пристигне, то се слага в тази опашка. При `receive` процесът блокира, ако няма съобщения в опашката
и чака докато поне едно съобщение не влезе в нея. Когато има такова съобщение, то се `pattern match`-ва
наред по условията изброени в блока на `receive` и ако има успех се изпълнява кодът съответстващ на това условие.

```elixir
pid = spawn(fn ->
  receive do
    pattern1 -> action1
    pattern2 -> action2
    ....
    patternN -> actionN
  end
end)

send(pid, pattern2)
```

Когато процесът изпълни логиката си, 'умира'.

Ето един истински пример:

```elixir
pid = spawn(fn ->
  receive do
    :say_hi -> IO.puts("Hi!")
    :say_by -> IO.puts("Bye!")
    {:say, name, msg} -> IO.puts([name, " says ", msg])
  end
end)

send(pid, {:say, "Arnold", "I'll be back!"})
```

Както казахме `receive` е като `case`, който се изпълнява върху полученото
съобщение. Съобщението може да е всякакъв тип. Можем да изпратим `PID`-а
на процеса, който извиква `send` и да го използваме за да получим отговор:

```elixir
pid = spawn(fn ->
  receive do
    {sender, :ping} when is_pid(sender) -> send(sender, {self(), :pong})
  end
end)

send(pid, {self(), :ping})
IO.puts("Let's wait for a pong!")

receive do
  {sender, :pong} when is_pid(sender) ->
    IO.puts([inspect(sender), " sends PONG!"])
end

# Ще се отпечата нещо като '#PID<0.150.0> sends PONG!'
```

Това е начинът по който два процеса могат да си говорят, с препращане на `PID`-овете си.
Със `self()` взимаме `PID`-а на текущия процес.
Така можем да имплементираме и синхронна комуникация - процес `A` изпраща съобщение на процес `B` и чака за отговор.

## Пример : паралелен `Enum.map`

Нека имаме числата от нула до петдесет. И нека поискаме да намерим квадратите им и
да ги върнем в списък. Това можем да направим така:

```elixir
1..50 |> Enum.map(fn x -> x * x end)
```

Нека сега направим нов `map`, който да изпълнява подадената си функция за всеки елемент в различен нов процес:

```elixir
defmodule PEnum do
  def map(enumerable, map_func) do
    enumerable |> Enum.map(spawn_func(map_func)) |> Enum.map(&receive_func/1)
  end

  defp spawn_func(map_func) do
    current_pid = self()
    fn x ->
      spawn(fn -> send(current_pid, {self(), map_func.(x)}) end)
    end
  end

  def receive_func(pid) do
    receive do
      {^pid, result} -> result
    end
  end
end
```

Дефинираме си нов модул - `PEnum`, подобно на `Enum`, той съдържа `map/2` функция.
Идеята на тази функция е за всеки елемент на подадената енумерация, да го трансформира, използвайки
нов процес, след което да върне трансформираните елементи в правилния ред.
Трансформацията се осъществява, с изпълнението на дадената като втори аргумент функция (`map_func`) върху елемента,
в новия процес.

Всъщност `PEnum.map/2`, използва `Enum.map/2`, подавайки ѝ енумерацията и функцията от по-висок ред `spawn_func`.
Функцията `spawn_func`
1. Приема функция и връща функция, която за дадена стойност `x` създава процес.
2. Този процес изпраща на адреса на процеса, извикващ `spawn_func`, `PID`-а си и резултата от функцията (`map_func`), подадена на `spawn_func`, изпълнена върху стойността `x` като параметър.

Това звучи сложно, но всъщност просто извикваме `Enum.map/2` с функция, която създава процес в който се изпълнява `map_func` и връща `PID`-а му.
И така на този етап имаме списък от `PID`-ове.

Отново използваме `Enum.map/2`, за да можем за всеки `PID`, да изчакаме съобщението от процеса адресиран в него.
Тези `PID`-ове идват в реда на създаването на процесите им, които пък са създадени по реда на елементите в енумерацията.
* За всеки такъв `PID`, чакаме за съобщение от него, което е във формата `{PID, <стойност>}`.
* Използваме `pin` оператора защото `pid` е променлива и не искаме да `match`-не всичко и да промени стойността си.
* Искаме да чакаме завършването на процесите по реда по който са създадени.
* Точно това прави `receive_func`.

И така, процесът извикващ `PEnum.map/2`, изчаква подред всеки процес, който създава.
В съобщенията се съдържа стойността от `map_func` извикана с елементите от енумерацията.

Ето пример:

```elixir
1..50 |> Enum.map(fn x -> :timer.sleep(1000); x* x end)
# След около 50 секунди ще имаме квадратите

1..50 |> PEnum.map(fn x -> :timer.sleep(1000); x* x end)
# След около секунда ще имаме квадратите
```

## Заключение

Процесите в `Elixir` са много леки и лесни за употреба, не споделят данни, които
могат да мутират и могат да се изпълняват паралелно. В следващата статия ще разберем
повече за тяхното устройство и начинът им на комуникация.
Край всичко това ще сравним приликите между `Elixir process` модела и `Actor` модела.