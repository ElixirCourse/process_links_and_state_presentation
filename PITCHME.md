#HSLIDE
## Връзки между процеси и състояние

#HSLIDE
![Image-Absolute](assets/title.png)

#HSLIDE
## Съдържание

1. Връзки между процеси
2. Наблюдение на процеси
3. Пазене на състояние в процеси
4. Агенти
5. Типове и поведения

#HSLIDE
## Връзки между процеси
![Image-Absolute](assets/chains.jpg)

#HSLIDE
### Какво става, когато процес 'умре'?
* Бива изчистен от паметта и просто спира да съществува.   <!-- .element: class="fragment" -->
* Никой не научава за това.  <!-- .element: class="fragment" -->

#HSLIDE
```elixir
defmodule Quitter do
  def run do
    Process.sleep(3000)
    exit(:i_am_tired)
  end
end
```

#HSLIDE
```elixir
pid = spawn(Quitter, :run, [])

# След няколко секунди
Process.alive?(pid) #false
```

#HSLIDE
![Image-Absolute](assets/quitter.jpg)

#HSLIDE
* Aко обаче ползваме `spawn_link`:

```elixir
spawn_link(Quitter, :run, [])
```

#HSLIDE
```elixir
spawn_link(Quitter, :run, [])

# След няколко секунди ще има грешка
```

#HSLIDE
```elixir
defmodule Quitter do
  def run_no_error do
    Process.sleep(3000)
  end
end

pid = spawn_link(Quitter, :run_no_error, [])

# След няколко секунди
Process.alive?(pid) #false
```

#HSLIDE
### spawn_link
* Ако някой от свързаните процеси излезе с грешка - всички свързани 'умират'.
* Ако някой от свързаните процеси излезе без грешка - другите не 'умират'.
* Връзките са двупосочни.


#HSLIDE
### Process.unlink
```elixir
pid = spawn_link(Quitter, :run, [])

Process.unlink(pid)

# Няма грешка
```

#HSLIDE
### Process.link
![Image-Absolute](assets/connection.jpg)

#HSLIDE
```elixir
defmodule Simple do
  def run do
    receive do
      {:link, pid} when is_pid(pid) ->
        Process.link(pid)
        run()
      :links ->
        IO.puts(inspect(Process.info(self(), :links)))
        run()
      :die ->
        IO.puts("#{inspect self()} is hit!")
        exit(:ouch)
    end
  end
end
```

#HSLIDE
```elixir
send(pid1, {:link, pid2}) # {:link, #PID<0.272.0>}
send(pid2, {:link, pid3}) # {:link, #PID<0.274.0>}
send(pid3, {:link, pid4}) # {:link, #PID<0.276.0>}
send(pid4, {:link, pid5}) # {:link, #PID<0.278.0>}
```

#HSLIDE
```elixir
send(pid1, :links) # {:links, [#PID<0.272.0>]}
send(pid2, :links) # {:links, [#PID<0.270.0>, #PID<0.274.0>]}
send(pid3, :links) # {:links, [#PID<0.272.0>, #PID<0.276.0>]}
send(pid4, :links) # {:links, [#PID<0.274.0>, #PID<0.278.0>]}
send(pid5, :links) # {:links, [#PID<0.276.0>]}
```

#HSLIDE
### Съобщения при грешка
![Image-Absolute](assets/unexpected-error.jpg)

#HSLIDE
* Тази грешка, която транзитивно убива свързаните процеси, е специално съобщение.
* Тези специални съобщения се наричат сигнали.  <!-- .element: class="fragment" -->
* 'Exit' сигналите са 'тайни' съобщения, които автоматично убиват процесите, които ги получат. <!-- .element: class="fragment" -->

#HSLIDE
* За да имаме 'fault-tolerant' система е добре да имаме способ да убиваме процесите ѝ
и да разбираме кога процес е бил ликвидиран.
* Тази fault-tolerant система трябва да има начин да рестартира процеси, когато те 'умрат'.

#HSLIDE
### Системни процеси
![Image-Absolute](assets/A-System-Administrator.jpg)

#HSLIDE
* В Elixir има специален вид процеси - системни процеси.
* Това са нормални процеси, които могат да трансформират 'exit' сигналите, които получават в нормални съобщения. <!-- .element: class="fragment" -->

#HSLIDE
```elixir
defmodule Quitter do
  def run do
    Process.sleep(3000)
    exit(:i_am_tired)
  end
end

Process.flag(:trap_exit, true)

spawn_link(Quitter, :run, [])
receive do msg -> IO.inspect(msg); end

# {:EXIT, #PID<0.95.0>, :i_am_tired}
```

#HSLIDE
### Грешки и поведение
![Image-Absolute](assets/shit.jpg)

#HSLIDE
```elixir
action = <?>
system_process = <?>

Process.flag(:trap_exit, system_process)
pid = spawn_link(action)

receive do
  msg -> IO.inspect(msg)
end
```

#HSLIDE
### Случай 1 :
При 'action = fn -> :nothing end'
1. При 'system_process = false' : Нищо. Текущият процес чака.
2. При 'system_process = true'  : Ще видим '{:EXIT, pid, :normal}'. Текущият процес продължава.

#HSLIDE
### Случай 2 :
При 'action = fn -> exit(:stuff) end'
1. При 'system_process = false' : Текущият процес умира.
2. При 'system_process = true'  : Ще видим '{:EXIT, pid, :stuff}'. Текущият процес продължава.

#HSLIDE
### Случай 3 :
При 'action = fn -> exit(:normal) end'
1. При 'system_process = false' : Нищо. Текущият процес чака.
2. При 'system_process = true'  : Ще видим '{:EXIT, pid, :normal}'. Текущият процес продължава.

#HSLIDE
### Случай 4 :
При 'action = fn -> raise("Stuff") end'
1. При 'system_process = false' : Текущият процес умира.
2. При 'system_process = true'  : Ще видим '{:EXIT, pid, {%RuntimeError{message: "Stuff"}....}'. Текущият процес продължава.

#HSLIDE
### Случай 5 :
При 'action = fn -> throw("Stuff") end'
1. При 'system_process = false' : Текущият процес умира.
2. При 'system_process = true'  : Ще видим '{:EXIT, pid, {{:nocatch, "Stuff"}, [...]}}'. Текущият процес продължава.

#HSLIDE
* Тези пет поведения покриват всички възможни случаи.
* Засега няма начин системен процес да бъде терминиран с какъвто и да е сигнал.

#HSLIDE
### Функцията 'Process.exit/2'
![Image-Absolute](assets/gun.jpg)

#HSLIDE
```elixir
defmodule HelloPrinter do
  def start_link do
    Process.sleep(5000)
    IO.puts("Hello")
    start_link()
  end
end
```

#HSLIDE
![Image-Absolute](assets/shut_up.jpg)

#HSLIDE
```elixir
Process.flag(:trap_exit, true)

pid = spawn_link(HelloPrinter, :start_link, [])
# Започва да печата "Hello" на всеки 5 секунди

Process.exit(pid, :spri_se_be)
# Спира да печата "Hello"

receive do
  msg -> IO.inspect(msg)
end
# {:EXIT, #PID<0.162.0>, :spri_se_be}
```

#HSLIDE
```elixir
defmodule HiPrinter do
  def start_link do
    Process.flag(:trap_exit, true)
    print()
  end

  defp print do
    receive do
      {:EXIT, pid, _} -> send(pid, "Няма да стане, батка!")
    after
      6000 -> IO.puts("Hi!")
    end
    print()
  end
end
```

#HSLIDE
```elixir
Process.exit(pid, :kill)
```

#HSLIDE
* Process.exit/2 с атома :kill, може да убие всякакъв процес, даже системен.
* Съобщението има статус :killed.

#HSLIDE
* Статусът от :kill стана :killed.
* Процесът, който изпрати :kill, беше свързан с този, който беше 'убит', и не искаме сигналът да рикошира обратно.
* Сигналът :kill не е транзитивен към връзките.

#HSLIDE
### Ликвидиране и поведение
![Image-Absolute](assets/terminator.jpg)

#HSLIDE
```elixir
action = <?>
system_process = <?>
status = <?>

Process.flag(:trap_exit, system_process)
pid = spawn_link(action)

Process.exit(pid, status)

receive do
  msg -> IO.inspect(msg)
end
```

#HSLIDE
### Случай 1 :
При 'action = fn -> Process.sleep(20_000) end' и 'status = :normal'
1. При 'system_process = false' : Нищо. Текущият процес чака.
2. При 'system_process = true'  : Нищо. Текущият процес чака.

#HSLIDE
* Процес не може да бъде 'убит' с Process.exit(pid, :normal)


#HSLIDE
### Случай 2 :
При 'action = fn -> Process.sleep(20_000) end' и 'status = :stuff'
1. При 'system_process = false' : Грешка. Текущият процес 'умира'.
2. При 'system_process = true'  : Получаваме съобщение '{'EXIT', pid, :stuff}'.

#HSLIDE
### Случай 3 :
При 'action = fn -> Process.sleep(20_000) end' и 'status = :kill'
1. При 'system_process = false' : Грешка. Текущият процес 'умира'.
2. При 'system_process = true'  : Получаваме съобщение '{'EXIT', pid, :killed}'.

#HSLIDE
### Случай 4 :
При 'action = fn -> Process.exit(self(), :kill) end' и 'status = <каквото-и-да-е>'
1. При 'system_process = false' : Грешка. Текущият процес 'умира'.
2. При 'system_process = true'  : Получаваме съобщение '{'EXIT', pid, :killed}'.

#HSLIDE
### Случай 5 :
При 'action = fn -> exit(:kill) end' и 'status = <каквото-и-да-е>'
1. При 'system_process = false' : Грешка. Текущият процес 'умира'.
2. При 'system_process = true'  : Получаваме съобщение '{'EXIT', pid, :kill}'. Странно, нали?

#HSLIDE
* exit(:reason) е различен от Process.exit(pid, :reason).
* exit(:reason) е нещо като throw, предизвиква 'хвърляне', което, ако не е хванато, 'убива' текущия процес.

#HSLIDE
* Когато два процеса са свързани, от тази грешка се произвежда сигнал със съобщение :reason, който ги 'убива'.
* Ако тази причина е :kill, тя се променя на :killed, за да не убива системни процеси, които са свързани.

#HSLIDE
* В случай 5 exit(:kill) може да бъде хванат с catch в процеса-източник.
* Не е хванат, затова процесът 'умира' с причина 'kill'.
* Свързаният системен процес получава сигнал 'killed' със съобщение оригиналната 'exit' причина - 'kill', а не 'kill'-сигнал.

#HSLIDE
## Наблюдение на процеси
![Image-Absolute](assets/watcher.gif)

#HSLIDE
* Свързването на процеси е двупосочно.
* Ако единият от тях 'умре', другият ще получи EXIT сигнал.  <!-- .element: class="fragment" -->
* Този сигнал ще 'убие' всички свързани процеси, ако те не са системни.  <!-- .element: class="fragment" -->

#HSLIDE
* Често искаме един процес да наблюдава друг без да му праща сигнал, ако случайно 'умре'.
* Също така искаме да наблюдаваме процеси без текущият процес да е системен.  <!-- .element: class="fragment" -->
* Ако те 'умрат', просто искаме да бъдем нотифицирани с нормална нотификация, за да предприемем нещо.  <!-- .element: class="fragment" -->

#HSLIDE
```elixir
pid = spawn(fn -> Process.sleep(3000) end)

Process.monitor(pid)

receive do
  msg -> IO.inspect(msg)
end
# След 3 секунди ще получим нещо такова
# {:DOWN, #Reference<...>, :process, #PID<...>, :normal}
```

#HSLIDE
* Добавяме монитор към процес с Process.monitor(pid).
* Тази фунцкия връща референция.  <!-- .element: class="fragment" -->
* Референциите са специален тип в Elixir, всяка от тях е уникална за текущия node.  <!-- .element: class="fragment" -->

#HSLIDE
* Когато получим DOWN съобщението, тази референция ще е вторият му елемент.
* Четвъртият е PID-а на процеса, който е завършил изпълнението си, а петият - статус.  <!-- .element: class="fragment" -->
* Това са същите тези статуси, които причиняваха сигналите при свързани процеси.  <!-- .element: class="fragment" -->

#HSLIDE
```elixir
pid = spawn(fn -> Process.sleep(3000) end)

ref = Process.monitor(pid)
Process.demonitor(ref)

receive do
  msg -> IO.inspect(msg)
after
  4000 -> IO.puts("Няма съобщения...")
end

# Ще видим 'Няма съобщения...'
```

#HSLIDE
* Има и версия на spawn, която създава нов процес и автоматично му добавя монитор:

```elixir
{pid, ref} = spawn_monitor(fn -> Process.sleep(3000) end)
Process.exit(pid, :kill)

receive do
  msg -> IO.inspect(msg)
end
# {:DOWN, <ref>, :process, pid, :killed
```

#HSLIDE
## Пазене на състояние в процеси

#HSLIDE
![Image-Absolute](assets/wrapped.jpg)


#HSLIDE
## Агенти
![Image-Absolute](assets/agents-agent-smithmatrix.jpg)

#HSLIDE
* Агентът е проста обвивка около състояние, съхранено в процес.
* Съхранението на състоянието е имплементирано чрез безкрайна рекурсия.

#HSLIDE
```elixir
{:ok, pid} = Agent.start_link(fn -> 5 end)
```

#HSLIDE
```elixir
Agent.get(pid, fn v -> v end)
# 5
```

#HSLIDE
## Типове и поведения
![Image-Absolute](assets/types-of-lighting.jpg)

#HSLIDE
### Типове
![Image-Absolute](assets/Tic-Tac-Toe-Cartoon-200.png)

#HSLIDE
### Поведения
![Image-Absolute](assets/rules.jpg)

#HSLIDE
* Поведенията дефинират множество от спецификации за функции, които даден модул, ако иска да изпълни даденото поведение, трябва да дефинира и имплементира.
* Компилаторът ще ни нотифицира с warning ако даден модул е дефинирал, че ще изпълни поведението, но някои от функциите не са дефинирани и имплементирани.

#HSLIDE
```elixir
defmodule SynchronousCallHandler do
  @type state :: term

  @callback init(args :: term) ::
    {:ok, state} |
    {:stop, reason :: term}

  @callback handle_call(request :: term, pid, state) ::
    {:reply, reply, new_state} |
    {:no_reply, new_state} |
    {:stop, reason} when reply: term, new_state: term, reason: term
end
```

#HSLIDE
### Полиморфизъм
![Image-Absolute](assets/cow.png)

#HSLIDE
_Жозе_ казва, че можем да разделим кода си на три групи:
* процеси  <!-- .element: class="fragment" -->
* модули  <!-- .element: class="fragment" -->
* данни  <!-- .element: class="fragment" -->

#HSLIDE
* Те са взаимносвързани.
* Процесите изпълняват код, който е дефиниран и структуриран в модули, които работят с дадени типове данни.

#HSLIDE
* Всяка от тези три категории има своят начин да постигне полиморфизъм:

#HSLIDE
* Можем да пратим едно и също съобщение на множество процеси.
* В зависимост от кода в процеса, ще имаме различно поведение.  <!-- .element: class="fragment" -->
* Не се интересуваме от процесите, важно е дали чакат за точно този тип съобщение.  <!-- .element: class="fragment" -->

#HSLIDE
* Когато викаме точно определена функция на модул, не се интересуваме какъв е модулът.
* Важно е да имплементира тази функция.  <!-- .element: class="fragment" -->

#HSLIDE
* Когато извикваме функция от протокол с аргумент даден тип - не се интересуваме от типа.
* Важното е, че функцията е имплементирана за него.  <!-- .element: class="fragment" -->

#HSLIDE
* Разликата между протоколите и поведенията е около какво се върти имплементацията - дали около тип данни или около модул.

#HSLIDE
#### Протоколи
* Kогато искаме някой да имплементира логика с определена форма за свой тип данни.
* Код, който искаме да бъде extend-нат за нов тип данни.  <!-- .element: class="fragment" -->

#HSLIDE
#### Поведения
* Когато имаме код работещ с набор от функции и искаме някой друг да може да имплементира тези функции.
* Когато пишем система, която може да се разширява с plugin-и.  <!-- .element: class="fragment" -->
* Когато пишем код, който може да сменя имплементации лесно.  <!-- .element: class="fragment" -->
* Когато самият ни код има места, които могат да се разширят от някой друг.  <!-- .element: class="fragment" -->

#HSLIDE
# Край
![Image-Absolute](assets/the-end.jpg)
