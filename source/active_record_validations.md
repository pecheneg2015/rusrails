Валидации Active Record
=======================

Это руководство научит, как осуществлять валидацию состояния объектов до того, как они будут направлены в базу данных, используя особенность валидаций Active Record.

После прочтения руководства вы узнаете:

* Как использовать встроенные хелперы валидации Active Record
* Как создавать свои собственные методы валидации
* Как работать с сообщениями об ошибках, генерируемыми в процессе валидации

Обзор валидаций
---------------

Вот пример очень простой валидации:

```ruby
class Person < ApplicationRecord
  validates :name, presence: true
end
```

```irb
irb> Person.create(name: "John Doe").valid?
=> true
irb> Person.create(name: nil).valid?
=> false
```

Как видите, наша валидация позволяет узнать, что наш `Person` не валиден без атрибута `name`. Второй `Person` не будет персистентным в базе данных.

Прежде чем погрузиться в подробности, давайте поговорим о том, как валидации вписываются в общую картину приложения.

### Зачем использовать валидации?

Валидации используются, чтобы быть уверенными, что только валидные данные сохраняются в вашу базу данных. Например, для вашего приложения может быть важно, что каждый пользователь предоставил валидный электронный и почтовый адреса. Валидации на уровне модели - наилучший способ убедиться, что в базу данных будут сохранены только валидные данные. Они не зависят от базы данных, не могут быть обойдены конечными пользователями и удобны в тестировании и обслуживании. Rails предоставляет встроенные хелперы для общих нужд, а также позволяет создавать свои собственные методы валидации.

Есть несколько способов валидации данных, прежде чем они будут сохранены в вашу базу данных, включая ограничения, встроенные в базу данных, валидации на клиентской части и валидации на уровне контроллера. Вкратце о плюсах и минусах:

* Ограничения базы данных и/или хранимые процедуры делают механизмы валидации зависимыми от базы данных, что делает тестирование и поддержку более трудными. Однако, если ваша база данных используется другими приложениями, валидация на уровне базы данных может безопасно обрабатывать некоторые вещи (такие как уникальность в нагруженных таблицах), которые затруднительно выполнять по-другому.
* Валидации на клиентской части могут быть очень полезны, но в целом ненадежны, если используются в одиночку. Если они используют JavaScript, они могут быть пропущены, если JavaScript отключен в клиентском браузере. Однако, если этот способ комбинировать с другими, валидации на клиентской части могут быть удобным способом предоставить пользователям немедленную обратную связь при использовании вашего сайта.
* Валидации на уровне контроллера заманчиво делать, но это часто приводит к громоздкости и трудности тестирования и поддержки. Во всех случаях, когда это возможно, держите свои контроллеры 'тощими', тогда с вашим приложением будет приятно работать в долгосрочной перспективе.

Выбирайте их под свои определенные специфичные задачи. Общее мнение команды Rails состоит в том, что валидации на уровне модели - наиболее подходящий вариант во многих случаях.

### Когда происходит валидация?

Есть два типа объектов Active Record: те, которые соответствуют строке в вашей базе данных, и те, которые нет. Когда создаете новый объект, например, используя метод `new`, этот объект еще не привязан к базе данных. Как только вы вызовете `save`, этот объект будет сохранен в подходящую таблицу базы данных. Active Record использует метод экземпляра `new_record?` для определения, есть ли уже объект в базе данных или нет. Рассмотрим следующий класс Active Record:

```ruby
class Person < ApplicationRecord
end
```

Можно увидеть, как он работает, взглянув на результат `bin/rails console`:

```irb
irb> p = Person.new(name: "John Doe")
=> #<Person id: nil, name: "John Doe", created_at: nil, updated_at: nil>

irb> p.new_record?
=> true

irb> p.save
=> true

irb> p.new_record?
=> false
```

Создание и сохранение новой записи посылает операцию SQL `INSERT` базе данных. Обновление существующей записи вместо этого посылает операцию SQL `UPDATE`. Валидации обычно запускаются до того, как эти команды посылаются базе данных. Если любая из валидаций проваливается, объект помечается как недействительный и Active Record не выполняет операцию `INSERT` или `UPDATE`. Это помогает избежать хранения невалидного объекта в базе данных. Можно выбирать запуск специфичных валидаций, когда объект создается, сохраняется или обновляется.

CAUTION: Есть разные методы изменения состояния объекта в базе данных. Некоторые методы вызывают валидации, некоторые нет. Это означает, что возможно сохранить в базу данных объект с недействительным статусом, если вы будете не внимательны.

Следующие методы вызывают валидацию, и сохраняют объект в базу данных только если он валиден:

* `create`
* `create!`
* `save`
* `save!`
* `update`
* `update!`

Версии с восклицательным знаком (т.е. `save!`) вызывают исключение, если запись недействительна. Невосклицательные версии не вызывают: `save` и `update` возвращают `false`, `create` возвращает объект.

### Пропуск валидаций

Следующие методы пропускают валидации, и сохраняют объект в базу данных, независимо от его валидности. Их нужно использовать осторожно.

* `decrement!`
* `decrement_counter`
* `increment!`
* `increment_counter`
* `insert`
* `insert!`
* `insert_all`
* `insert_all!`
* `toggle!`
* `touch`
* `touch_all`
* `update_all`
* `update_attribute`
* `update_column`
* `update_columns`
* `update_counters`
* `upsert`
* `upsert_all`

Заметьте, что `save` также имеет способность пропустить валидации, если передать `validate: false` как аргумент. Этот способ нужно использовать осторожно.

* `save(validate: false)`

### `valid?` или `invalid?`

Перед сохранением объекта Active Record, Rails запускает ваши валидации. Если валидации производят какие-либо ошибки, Rails не сохраняет этот объект.

Вы также можете запускать эти валидации самостоятельно. [`valid?`][] вызывает ваши валидации и возвращает true, если ни одной ошибки не было найдено у объекта, иначе false.

```ruby
class Person < ApplicationRecord
  validates :name, presence: true
end
```

```irb
irb> Person.create(name: "John Doe").valid?
=> true
irb> Person.create(name: nil).valid?
=> false
```

После того, как Active Record выполнит валидации, все неудачи будут доступны в методе экземпляра [`errors`][], возвращающем коллекцию ошибок. По определению объект валиден, если эта коллекция будет пуста после запуска валидаций.

Заметьте, что объект, созданный с помощью `new` не сообщает об ошибках, даже если технически невалиден, поскольку валидации автоматически запускаются только когда сохраняется объект, как в случае с методами `create` или `save`.

```ruby
class Person < ApplicationRecord
  validates :name, presence: true
end
```

```irb
irb> p = Person.new
=> #<Person id: nil, name: nil>
irb> p.errors.size
=> 0

irb> p.valid?
=> false
irb> p.errors.objects.first.full_message
=> "Name can’t be blank"

irb> p = Person.create
=> #<Person id: nil, name: nil>
irb> p.errors.objects.first.full_message
=> "Name can’t be blank"

irb> p.save
=> false

irb> p.save!
ActiveRecord::RecordInvalid: Validation failed: Name can’t be blank

irb> Person.create!
ActiveRecord::RecordInvalid: Validation failed: Name can’t be blank
```

[`invalid?`][] это антипод `valid?`. Он запускает ваши валидации, возвращая true, если для объекта были добавлены ошибки, и false в противном случае.

[`errors`]: https://api.rubyonrails.org/classes/ActiveModel/Validations.html#method-i-errors
[`invalid?`]: https://api.rubyonrails.org/classes/ActiveModel/Validations.html#method-i-invalid-3F
[`valid?`]: https://api.rubyonrails.org/classes/ActiveRecord/Validations.html#method-i-valid-3F

### `errors[]`

Чтобы проверить, является или нет конкретный атрибут объекта валидным, можно использовать [`errors[:attribute]`][Errors#squarebrackets], который возвращает массив со всеми сообщениями об ошибке атрибута, когда нет ошибок по определенному атрибуту, возвращается пустой массив.

Этот метод полезен только _после того_, как валидации были запущены, так как он всего лишь исследует коллекцию errors, но сам не вызывает валидации. Он отличается от метода `ActiveRecord::Base#invalid?`, описанного выше, тем, что не проверяет валидность объекта в целом. Он всего лишь проверяет, какие ошибки были найдены для отдельного атрибута объекта.

```ruby
class Person < ApplicationRecord
  validates :name, presence: true
end
```

```irb
irb> Person.new.errors[:name].any?
=> false
irb> Person.create.errors[:name].any?
=> true
```

Мы рассмотрим ошибки валидации подробнее в разделе [Работаем с ошибками валидации](#working-with-validation-errors).

[Errors#squarebrackets]: https://api.rubyonrails.org/classes/ActiveModel/Errors.html#method-i-5B-5D

Валидационные хелперы
---------------------

Active Record предлагает множество предопределенных валидационных хелперов, которые вы можете использовать прямо внутри ваших определений класса. Эти хелперы предоставляют общие правила валидации. Каждый раз, когда валидация проваливается, ошибка добавляется в коллекцию `errors` объекта и связывается с атрибутом, который подлежал валидации.

Каждый хелпер принимает произвольное количество имен атрибутов, поэтому в одной строчке кода можно добавить валидации одинакового вида для нескольких атрибутов.

Они все принимают опции `:on` и `:message`, которые определяют, когда валидация должна быть запущена, и какое сообщение должно быть добавлено в коллекцию `errors`, если она провалится. Опция `:on` принимает одно из значений `:create` или `:update`. Для каждого валидационного хелпера есть свое сообщение об ошибке по умолчанию. Эти сообщения используются, если не определена опция `:message`. Давайте рассмотрим каждый из доступных хелперов.

INFO: Чтобы просмотреть список доступных хелперов по умолчанию, обратитесь к [`ActiveModel::Validations::HelperMethods`][].

[`ActiveModel::Validations::HelperMethods`]: https://api.rubyonrails.org/classes/ActiveModel/Validations/HelperMethods.html

### `acceptance`

Этот метод проверяет, что чекбокс в пользовательском интерфейсе был нажат, когда форма была подтверждена. Обычно используется, когда пользователю нужно согласиться с условиями использования вашего приложения, подтвердить, что некоторый текст прочтен, или другой подобной концепции.

```ruby
class Person < ApplicationRecord
  validates :terms_of_service, acceptance: true
end
```

Эта проверка выполнится, только если `terms_of_service` не `nil`. Для этого хелпера сообщение об ошибке по умолчанию следующее _"must be accepted"_. Можно передать произвольное сообщение с помощью опции `message`.

```ruby
class Person < ApplicationRecord
  validates :terms_of_service, acceptance: { message: 'must be abided' }
end
```

Также он может получать опцию `:accept`, которая определяет допустимые значения, которые будут считаться приемлемыми. По умолчанию это `['1', true]`, но его можно изменить.

```ruby
class Person < ApplicationRecord
  validates :eula, acceptance: { accept: ['TRUE', 'accepted'] }
end
```

Эта валидация очень специфична для веб-приложений, и ее принятие не нужно записывать куда-либо в базу данных. Если у вас нет поля для него, хелпер создаст виртуальный атрибут. Если поле существует в базе данных, опция `accept` должна быть установлена или включать `true`, а иначе эта валидация не будет выполнена.

### `confirmation`

Этот хелпер можно использовать, если у вас есть два текстовых поля, из которых нужно получить полностью идентичное содержание. Например, вы хотите подтверждение адреса электронной почты или пароля. Эта валидация создает виртуальный атрибут, имя которого равно имени подтверждаемого поля с добавлением "\_confirmation".

```ruby
class Person < ApplicationRecord
  validates :email, confirmation: true
end
```

В вашем шаблоне вью нужно использовать что-то вроде этого

```erb
<%= text_field :person, :email %>
<%= text_field :person, :email_confirmation %>
```

NOTE: Эта проверка выполняется, только если `email_confirmation` не равно `nil`. Чтобы требовать подтверждение, нужно добавить еще проверку на существование проверяемого атрибута (мы рассмотрим `presence` [чуть позже](#presence)):

```ruby
class Person < ApplicationRecord
  validates :email, confirmation: true
  validates :email_confirmation, presence: true
end
```

Также имеется опция `:case_sensitive`, которую используют, чтобы определить, должно ли ограничение подтверждения быть чувствительным к регистру. Эта опция по умолчанию true.

```ruby
class Person < ApplicationRecord
  validates :email, confirmation: { case_sensitive: false }
end
```

По умолчанию сообщение об ошибке для этого хелпера такое _"doesn't match confirmation"_. Также можно передать произвольное сообщение с помощью опции `message`.

В основном, при использовании этого валидатора, хочется объединить его с опцией `:if`, чтобы валидировать поле "_confirmation" только тогда, когда изменяется изначальное поле, а **не** каждый раз при сохранении записи. Подробности об [условных валидациях](#conditional-validation) позже.

```ruby
class Person < ApplicationRecord
  validates :email, confirmation: true
  validates :email_confirmation, presence: true, if: :email_changed?
end
```

### `comparison`

Эта проверка проводит валидацию сравнения между любыми двумя сравниваемыми значениями. Этот валидатор требует предоставления опции сравнения.

```ruby
class Promotion < ApplicationRecord
  validates :end_date, comparison: { greater_than: :start_date }
end
```

Сообщение об ошибке по умолчанию для этого хелпера _"failed comparison"_. Также можно передать произвольное сообщение с помощью опции `message`.

Все эти опции поддерживаются:

* `:greater_than` - Указывает что значение должно быть больше, чем предоставленное значение. Сообщение об ошибке для этой опции такое _"must be greater than %{count}"_.
* `:greater_than_or_equal_to` - Указывает что значение должно быть больше или равно предоставленному значению. Сообщение об ошибке для этой опции такое _"must be greater than or equal to %{count}"_.
* `:equal_to` - Указывает что значение должно быть равно предоставленному значению. Сообщение об ошибке для этой опции такое _"must be equal to %{count}"_.
* `:less_than` - Указывает что значение должно быть меньше, чем предоставленное значение. Сообщение об ошибке для этой опции такое _"must be less than %{count}"_.
* `:less_than_or_equal_to` - Указывает что значение должно быть меньше или равно предоставленному значению. Сообщение об ошибке для этой опции такое _"must be less than or equal to %{count}"_.
* `:other_than` - Указывает что значение должно быть иным, чем предоставленное значение. Сообщение об ошибке для этой опции такое _"must be other than %{count}"_.

NOTE: Этот валидатор требует предоставления опции сравнения. Каждая опция принимает значение, proc или символ. Любой класс, включающий Comparable, может быть сравнен.

### `format`

Этот хелпер проводит валидацию значений атрибутов, тестируя их на соответствие указанному регулярному выражению, которое определяется с помощью опции `:with`.

```ruby
class Product < ApplicationRecord
  validates :legacy_code, format: { with: /\A[a-zA-Z]+\z/,
    message: "only allows letters" }
end
```

В качестве альтернативы можно потребовать, чтобы указанный атрибут _не_ соответствовал регулярному выражению, используя опцию `:without`.

Наоборот, используя опцию `:without`, можно потребовать, что указанный атрибут _не_ соответствует регулярному выражению.

В любом случае, предоставленная опция `:with` или `:without` должна быть либо регулярное выражение, либо proc, либо lambda, возвращающая это.

Значение сообщения об ошибке по умолчанию "_is invalid_".

WARNING. Используйте `\A` и `\z` для соответствия началу и концу строки, `^` и `$` для соответствия началу/концу строчки. Из-за частого неправильного использования `^` и `$`, необходимо передать опцию `multiline: true` в случае использования этих двух якорей в предоставленном регулярном выражении. В большинстве случаев следует использовать `\A` и `\z`.

### `inclusion`

Этот хелпер проводит валидацию значений атрибутов на включение в указанный набор. Фактически этот набор может быть любым перечисляемым объектом.

```ruby
class Coffee < ApplicationRecord
  validates :size, inclusion: { in: %w(small medium large),
    message: "%{value} is not a valid size" }
end
```

Хелпер `inclusion` имеет опцию `:in`, которая получает набор значений, которые должны быть приняты. Опция `:in` имеет псевдоним `:within`, который используется для тех же целей. Предыдущий пример использует опцию `:message`, чтобы показать вам, как можно включать значение атрибута. Для того, чтобы увидеть все опции смотрите [документацию по message](#message).

Значение сообщения об ошибке по умолчанию для этого хелпера такое "_is not included in the list_".

### `exclusion`

Противоположностью `inclusion` является... `exclusion`!

Этот хелпер проводит валидацию того, что значения атрибутов не включены в указанный набор. Фактически, этот набор может быть любым перечисляемым объектом.

```ruby
class Account < ApplicationRecord
  validates :subdomain, exclusion: { in: %w(www us ca jp),
    message: "%{value} is reserved." }
end
```

Хелпер `exclusion` имеет опцию `:in`, которая получает набор значений, которые не должны приниматься проверяемыми атрибутами. Опция `:in` имеет псевдоним `:within`, который используется для тех же целей. Этот пример использует опцию `:message`, чтобы показать вам, как можно включать значение атрибута. Для того, чтобы увидеть все опции аргументов сообщения смотрите [документацию по message](#message).

Значение сообщения об ошибке по умолчанию "_is reserved_".

Альтернативно к традиционным перечисляемым (наподобие Array), можно предоставить proc, lambda или symbol, возвращающие перечисляемое. Если перечисляемое является числовым, временным или дата-временным рядом, проверка осуществляется с помощью `Range#cover?`, в противном случае `include?`. при использовании proc или lambda, проверяемый экземпляр передается как аргумент.

### `length`

Этот хелпер проводит валидацию длины значений атрибутов. Он предлагает ряд опций, с помощью которых вы можете определить ограничения по длине разными способами:

```ruby
class Person < ApplicationRecord
  validates :name, length: { minimum: 2 }
  validates :bio, length: { maximum: 500 }
  validates :password, length: { in: 6..20 }
  validates :registration_number, length: { is: 6 }
end
```

Возможные опции ограничения длины такие:

* `:minimum` - атрибут не может быть меньше определенной длины.
* `:maximum` - атрибут не может быть больше определенной длины.
* `:in` (или `:within`) - длина атрибута должна находиться в указанном интервале. Значение этой опции должно быть интервалом.
* `:is` - длина атрибута должна быть равной указанному значению.

Значение сообщения об ошибке по умолчанию зависит от типа выполняемой валидации длины. Можно настроить эти сообщения, используя опции `:wrong_length`, `:too_long` и `:too_short`, и `%{count}` как местозаполнитель (placeholder) числа, соответствующего длине используемого ограничения. Можете использовать опцию `:message` для определения сообщения об ошибке.

```ruby
class Person < ApplicationRecord
  validates :bio, length: { maximum: 1000,
    too_long: "%{count} characters is the maximum allowed" }
end
```

Отметьте, что сообщения об ошибке по умолчанию во множественном числе (т.е., "is too short (minimum is %{count} characters)"). По этой причине, когда `:minimum` равно 1, следует предоставить собственное сообщение или использовать вместо него `presence: true`. Когда `:in` или `:within` имеют как нижнюю границу 1, следует или предоставить собственное сообщение, или вызвать `presence` перед `length`.

NOTE: Одновременно может быть использована одна опция, кроме опций `:minimum` и `:maximum`, которые комбинируются вместе.

### `numericality`

Этот хелпер проводит валидацию того, что ваши атрибуты имеют только числовые значения. По умолчанию, этому будет соответствовать возможный знак первым символом, и следующее за ним целочисленное или с плавающей запятой число.

Чтобы определить, что допустимы только целочисленные значения, установите `:only_integer` в true. Тогда будет использоваться регулярное выражение для проведения валидации значения атрибута.

```ruby
/\A[+-]?\d+\z/
```

В противном случае, он будет пытаться конвертировать значение в число, используя `Float`. `Float` приводятся к `BigDecimal` с использованием значения precision столбца или максимум 15 разрядов.

```ruby
class Player < ApplicationRecord
  validates :points, numericality: true
  validates :games_played, numericality: { only_integer: true }
end
```

Для `:only_integer` значение сообщения об ошибке по умолчанию _"must be an integer"_.

Кроме `:only_integer`, хелпер `validates_numericality_of` также принимает опцию `:only_numeric`, которая указывает, что значение должно быть экземпляром `Numeric` и пытается парсить значение, если оно `String`.

NOTE: По умолчанию `numericality` не допускает значения `nil`. Чтобы их разрешить, можно использовать опцию `allow_nil: true`. Отметьте, что для столбцов `Integer` и `Float` пустые строки конвертируются в `nil`.

По умолчанию сообщение об ошибке _"is not a number"_.

Также есть множество опций для добавления ограничений к приемлемым значениям:

* `:greater_than` - Указывает, что значение должно быть больше, чем значение опции. По умолчанию сообщение об ошибке для этой опции такое _"must be greater than %{count}"_.
* `:greater_than_or_equal_to` - Указывает, что значение должно быть больше или равно значению опции. По умолчанию сообщение об ошибке для этой опции такое _"must be greater than or equal to %{count}"_.
* `:equal_to` - Указывает, что значение должно быть равно значению опции. По умолчанию сообщение об ошибке для этой опции такое _"must be equal to %{count}"_.
* `:less_than` - Указывает, что значение должно быть меньше, чем значение опции. По умолчанию сообщение об ошибке для этой опции такое _"must be less than %{count}"_.
* `:less_than_or_equal_to` - Указывает, что значение должно быть меньше или равно значению опции. По умолчанию сообщение об ошибке для этой опции такое _"must be less than or equal to %{count}"_.
* `:other_than` - Указывает, что значение должно отличаться от представленного значения. По умолчанию сообщение об ошибке для этой опции такое _"must be other than %{count}"_.
* `:in` - Указывает, что значение должно быть в предоставленном диапазоне. По умолчанию сообщение об ошибке для этой опции такое _"must be in %{count}"_.
* `:odd` - Указывает, что значение должно быть нечетным. По умолчанию сообщение об ошибке для этой опции такое _"must be odd"_.
* `:even` - Указывает, что значение должно быть четным. По умолчанию сообщение об ошибке для этой опции такое _"must be even"_.

### `presence`

Этот хелпер проводит валидацию того, что определенные атрибуты не пустые. Он использует метод [`Object#blank?`][] для проверки того, является ли значение или `nil`, или пустой строкой (это строка, которая или пуста, или состоит из пробелов).

```ruby
class Person < ApplicationRecord
  validates :name, :login, :email, presence: true
end
```

Если хотите быть уверенным, что связь существует, нужно проверить, существует ли сам связанный объект, а не внешний ключ, используемый для связи.

```ruby
class Supplier < ApplicationRecord
  has_one :account
  validates :account, presence: true
end
```

Для того, чтобы проверять связанные записи, чье присутствие необходимо, нужно определить опцию `:inverse_of` для связи:

```ruby
class Order < ApplicationRecord
  has_many :line_items, inverse_of: :order
end
```

NOTE: Если хотите убедиться, что связь и существует, и валидна, нужно также использовать `validates_associated`. Подробнее [ниже](#validates-associated)

При проведении валидации существования объекта, связанного отношением `has_one` или `has_many`, будет проверено, что объект ни `blank?`, ни `marked_for_destruction?`.

Так как `false.blank?` это true, если хотите провести валидацию существования булева поля, нужно использовать одну из следующих валидаций:

```ruby
# Значение _должно_ быть true или false
validates :boolean_field_name, inclusion: [true, false]
# Значение _не должно_ быть nil, только true или false
validates :boolean_field_name, exclusion: [nil]
```

При использовании одной из этих валидаций, вы можете быть уверены, что значение не будет `nil`, которое в большинстве случаев преобразуется в `NULL` значение.

Значение об ошибке по умолчанию _"can’t be blank"_.

[`Object#blank?`]: https://api.rubyonrails.org/classes/Object.html#method-i-blank-3F

### `absence`

Этот хелпер проверяет, что указанные атрибуты отсутствуют. Он использует метод [`Object#present?`][] для проверки, что значение является либо nil, либо пустой строкой (то есть либо нулевой длины, либо состоящей из пробелов).

```ruby
class Person < ApplicationRecord
  validates :name, :login, :email, absence: true
end
```

Если хотите убедиться, что отсутствует связь, необходимо проверить, что отсутствует сам связанный объект, а не внешний ключ, используемый для связи.

```ruby
class LineItem < ApplicationRecord
  belongs_to :order
  validates :order, absence: true
end
```

Чтобы проверять связанные объекты, отсутствие которых требуется, для связи необходимо указать опцию `:inverse_of`:

```ruby
class Order < ApplicationRecord
  has_many :line_items, inverse_of: :order
end
```

NOTE: Если желаете убедиться, что связь и существует, и валидна, также необходимо использовать `validates_associated`. Подробнее [ниже](#validates-associated)

Если проверяете отсутствие объекта, связанного отношением `has_one` или `has_many`, он проверит, что объект и не `present?`, и не `marked_for_destruction?`.

Поскольку `false.present?` является false, если хотите проверить отсутствие булева поля, следует использовать `validates :field_name, exclusion: { in: [true, false] }`.

По умолчанию сообщение об ошибке _"must be blank"_.

[`Object#present?`]: https://api.rubyonrails.org/classes/Object.html#method-i-present-3F

### `uniqueness`

Этот хелпер проводит валидацию того, что значение атрибута уникально, перед тем, как объект будет сохранен.

```ruby
class Account < ApplicationRecord
  validates :email, uniqueness: true
end
```

Валидация производится путем SQL-запроса в таблицу модели, поиска существующей записи с тем же значением атрибута.

Имеется опция `:scope`, которую можно использовать для определения одного и более атрибутов, используемых для ограничения проверки уникальности:

```ruby
class Holiday < ApplicationRecord
  validates :name, uniqueness: { scope: :year,
    message: "should happen once per year" }
end
```

WARNING. Эта валидация не создает условие уникальности в базе данных, следовательно, может произойти так, что два разных подключения к базе данных создадут две записи с одинаковым значением для столбца, который вы подразумеваете уникальным. Чтобы этого избежать, нужно создать индекс unique на оба столбцах в вашей базе данных.

Чтобы добавить ограничение уникальности для базы данных, используйте выражение [`add_index`][] в миграции и включите опцию `unique: true`.

Если хотите создать ограничение на уровне базы данных, чтобы предотвратить возможные нарушения валидации уникальности с помощью опции `:scope`, необходимо создать индекс уникальности на обоих столбцах базы данных. Подробнее об индексах для нескольких столбцов смотрите в [мануале MySQL][], или примеры ограничений уникальности, относящихся к группе столбцов в [мануале PostgreSQL][].

Также имеется опция `:case_sensitive`, которой можно определить, будет ли ограничение уникальности чувствительно к регистру, не чувствительным к регистру или соответствовать сортировке базы данных по умолчанию. Опцией по умолчанию является соответствие сортировке базы данных по умолчанию.

```ruby
class Person < ApplicationRecord
  validates :name, uniqueness: { case_sensitive: false }
end
```

WARNING. Отметьте, что некоторые базы данных настроены на выполнение чувствительного к регистру поиска в любом случае.

Имеется опция `:conditions`, которой можно указать дополнительные условия в виде фрагмента SQL `WHERE` для ограничения поиска уникального ограничения (напр. `conditions: -> { where(status: 'active') }`).

По умолчанию сообщение об ошибке _"has already been taken"_.

Подробнее смотрите [`validates_uniqueness_of`][].

[`validates_uniqueness_of`]: https://api.rubyonrails.org/classes/ActiveRecord/Validations/ClassMethods.html#method-i-validates_uniqueness_of
[`add_index`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_index
[the MySQL manual]: https://dev.mysql.com/doc/refman/en/multiple-column-indexes.html
[the PostgreSQL manual]: https://www.postgresql.org/docs/current/static/ddl-constraints.html

### `validates_associated`

Этот хелпер можно использовать, когда у вашей модели есть связи, которые всегда нужно проверять на валидность. Каждый раз, когда вы пытаетесь сохранить свой объект, будет вызван метод `valid?` для каждого из связанных объектов.

```ruby
class Library < ApplicationRecord
  has_many :books
  validates_associated :books
end
```

Эта валидация работает со всеми типами связей.

CAUTION: Не используйте `validates_associated` на обоих концах ваших связей, они будут вызывать друг друга в бесконечном цикле.

Для [`validates_associated`][] сообщение об ошибке по умолчанию следующее _"is invalid"_. Заметьте, что каждый связанный объект имеет свою собственную коллекцию `errors`; ошибки не добавляются к вызывающей модели.

NOTE: [`validates_associated`][] можно использовать только с объектами ActiveRecord, а все до этого можно было также использовать с любым объектом, включающим [`ActiveModel::Validations`][].

[`validates_associated`]: https://api.rubyonrails.org/classes/ActiveRecord/Validations/ClassMethods.html#method-i-validates_associated

### `validates_each`

Этот хелпер валидирует атрибуты в блоке. У него нет предопределенной функции валидации. Его следует создавать с помощью блока, и каждый атрибут, переданный в [`validates_each`][] будет в нем протестирован.

В следующем примере мы отвергаем имена и фамилии, начинающиеся со строчной буквы.

```ruby
class Person < ApplicationRecord
  validates_each :name, :surname do |record, attr, value|
    record.errors.add(attr, 'must start with upper case') if /\A[[:lower:]]/.match?(value)
  end
end
```

Блок получает запись, имя атрибута и значение атрибута.

Можно делать все, что угодно, чтобы проверять валидность данных внутри блока. Если ваша валидация проваливается, следует добавить ошибку к модели, что сделает ее невалидной.

[`validates_each`]: https://api.rubyonrails.org/classes/ActiveModel/Validations/ClassMethods.html#method-i-validates_each

### `validates_with`

Этот хелпер передает запись в отдельный класс для валидации.

```ruby
class GoodnessValidator < ActiveModel::Validator
  def validate(record)
    if record.first_name == "Evil"
      record.errors.add :base, "This person is evil"
    end
  end
end

class Person < ApplicationRecord
  validates_with GoodnessValidator
end
```

Для `validates_with` нет сообщения об ошибке по умолчанию. Следует вручную добавлять ошибки в коллекцию errors записи в классе валидатора.

NOTE: Ошибки, добавляемые в `record.errors[:base]` относятся к состоянию записи в целом.

Для реализации метода валидации, необходимо принимать параметр `record`, который является записью, проходящей валидацию.

Если хотите добавить ошибку к определенному атрибуту, передайте его в качестве первого аргумента, как в `record.errors.add(:first_name, "please choose another name")`. Мы раскроем [валидационные ошибки][] гораздо детальнее позже.

```ruby
def validate(record)
  if record.some_field != "acceptable"
    record.errors.add :some_field, "this field is unacceptable"
  end
end
```

Хелпер [`validates_with`][] принимает класс или список классов для использования в валидации.

```ruby
class Person < ApplicationRecord
  validates_with MyValidator, MyOtherValidator, on: :create
end
```

Подобно всем другим валидациям, `validates_with` принимает опции `:if`, `:unless` и `:on`. Если передадите любые другие опции, они будут переданы в класс валидатора как `options`:

```ruby
class GoodnessValidator < ActiveModel::Validator
  def validate(record)
    if options[:fields].any? { |field| record.send(field) == "Evil" }
      record.errors.add :base, "This person is evil"
    end
  end
end

class Person < ApplicationRecord
  validates_with GoodnessValidator, fields: [:first_name, :last_name]
end
```

Отметьте, что валидатор будет инициализирован **только один раз** на протяжении всего жизненного цикла приложения, а не при каждом запуске валидации, поэтому будьте аккуратнее с использованием переменных экземпляра в нем.

Если ваш валидатор настолько сложный, что вы хотите использовать переменные экземпляра, вместо него проще использовать обычные объекты Ruby:

```ruby
class Person < ApplicationRecord
  validate do |person|
    GoodnessValidator.new(person).validate
  end
end

class GoodnessValidator
  def initialize(person)
    @person = person
  end

  def validate
    if some_complex_condition_involving_ivars_and_private_methods?
      @person.errors.add :base, "This person is evil"
    end
  end

  # ...
end
```

Мы раскроем [произвольные валидации](#performing-custom-validations) позже.

[валидационные ошибки](#working-with-validation-errors)
[`validates_with`]: https://api.rubyonrails.org/classes/ActiveModel/Validations/ClassMethods.html#method-i-validates_with

Общие опции валидаций
---------------------

Есть несколько общих опций валидаций, поддерживаемых валидаторами, которые мы только что описали, давайте пройдемся по ним!

NOTE: Не все из этих опций поддерживаются каждым валидатором, обратитесь к документации API для [`ActiveModel::Validations`][].

Используя любой из упомянутых методов валидации, также имеется список общих для валидаторов опций. Сейчас мы их раскроем!

* [`:allow_nil`](#allow-nil): Пропускает валидацию, если атрибут `nil`.
* [`:allow_blank`](#allow-blank): Пропускает валидацию, если атрибут пустой.
* [`:message`](#message): Указывает произвольное сообщение об ошибке.
* [`:on`](#on): Указывает контексты, в которых валидация активна.
* [`:strict`](#strict-validations): Вызывает исключение, когда валидация проваливается.
* [`:if` и `:unless`](#conditional-validation): Указывает, когда валидации следует или не следует происходить.

[`ActiveModel::Validations`]: https://api.rubyonrails.org/classes/ActiveModel/Validations.html

### `:allow_nil`

Опция `:allow_nil` пропускает валидацию, когда проверяемое значение равно `nil`.

```ruby
class Coffee < ApplicationRecord
  validates :size, inclusion: { in: %w(small medium large),
    message: "%{value} is not a valid size" }, allow_nil: true
end
```

```irb
irb> Coffee.create(size: nil).valid?
=> true
irb> Coffee.create(size: "mega").valid?
=> false
```

Для того, чтобы увидеть все опции аргументов сообщения смотрите [документацию по message](#message).

### `:allow_blank`

Опция `:allow_blank` подобна опции `:allow_nil`. Эта опция пропускает валидацию, если значение атрибута `blank?`, например `nil` или пустая строка.

```ruby
class Topic < ApplicationRecord
  validates :title, length: { is: 5 }, allow_blank: true
end
```

```irb
irb> Topic.create(title: "").valid?
=> true
irb> Topic.create(title: nil).valid?
=> true
```

### `:message`

Как мы уже видели, опция `:message` позволяет определить сообщение, которое будет добавлено в коллекцию `errors`, когда валидация проваливается. Если эта опция не используется, Active Record будет использовать соответствующие сообщения об ошибках по умолчанию для каждого валидационного хелпера.

Опция `:message` принимает `String` или `Proc` в качестве значения.

Значение `String` в `:message` может опционально содержать любые из `%{value}`, `%{attribute}` и `%{model}`, которые будут динамически заменены, когда валидация провалится. Эта замена выполняется, если используется гем i18n, и местозаполнитель должен полностью совпадать, пробелы не допускаются.

```ruby
class Person < ApplicationRecord
  # Жестко закодированное сообщение
  validates :name, presence: { message: "must be given please" }

  # Сообщение со значением с динамическим атрибутом. %{value} будет заменено
  # фактическим значением атрибута. Также доступны %{attribute} и %{model}.
  validates :age, numericality: { message: "%{value} seems wrong" }
```

Значение `Proc` в `:message` задается с двумя аргументами: проверяемым объектом и хэшем с ключами `:model`, `:attribute` и `:value`.

```ruby
class Person < ApplicationRecord  validates :username,
    uniqueness: {
      # object = person object being validated
      # data = { model: "Person", attribute: "Username", value: <username> }
      message: ->(object, data) do
        "Hey #{object.name}, #{data[:value]} is already taken."
      end
    }
end
```

### `:on`

Опция `:on` позволяет определить, когда должна произойти валидация. Стандартное поведение для всех встроенных валидационных хелперов это запускаться при сохранении (и когда создается новая запись, и когда она обновляется). Если хотите изменить это, используйте `on: :create`, для запуска валидации только когда создается новая запись, или `on: :update`, для запуска валидации когда запись обновляется.

```ruby
class Person < ApplicationRecord
  # будет возможно обновить email с дублирующим значением
  validates :email, uniqueness: true, on: :create

  # будет возможно создать запись с нечисловым возрастом
  validates :age, numericality: true, on: :update

  # по умолчанию (проверяет и при создании, и при обновлении)
  validates :name, presence: true
end
```

`on:` также можно использовать для определения пользовательского контекста. Пользовательские контексты должны быть явно включены с помощью передачи имени контекста в `valid?`, `invalid?` или `save`.

```ruby
class Person < ApplicationRecord
  validates :email, uniqueness: true, on: :account_setup
  validates :age, numericality: true, on: :account_setup
end
```

```irb
irb> person = Person.new(age: 'thirty-three')
irb> person.valid?
=> true
irb> person.valid?(:account_setup)
=> false
irb> person.errors.messages
=> {:email=>["has already been taken"], :age=>["is not a number"]}
```

`person.valid?(:account_setup)` выполнит обе валидации без сохранения модели. `person.save(context: :account_setup)` перед сохранением валидирует `person` в контексте `account_setup`.

Передача массива символов также приемлема.

```ruby
class Book
  include ActiveModel::Validations

  validates :title, presence: true, on: [:update, :ensure_title]
end
```

```irb
irb> book = Book.new(title: nil)
irb> book.valid?
=> true
irb> book.valid?(:ensure_title)
=> false
irb> book.errors.messages
=> {:title=>["can’t be blank"]}
```

При вызове с явным контекстом, будут запущены валидации не только этого контекста, но и валидации _без_ контекста.

```ruby
class Person < ApplicationRecord
  validates :email, uniqueness: true, on: :account_setup
  validates :age, numericality: true, on: :account_setup
  validates :name, presence: true
end
```

```irb
irb> person = Person.new
irb> person.valid?(:account_setup)
=> false
irb> person.errors.messages
=> {:email=>["has already been taken"], :age=>["is not a number"], :name=>["can’t be blank"]}
```

Мы раскроем больше способов использования для `on:` в [руководстве по колбэкам](/active-record-callbacks).

(strict-validations) Строгие валидации
-----------------

Также можно определить валидации строгими, чтобы они вызывали `ActiveModel::StrictValidationFailed`, когда объект невалиден.

```ruby
class Person < ApplicationRecord
  validates :name, presence: { strict: true }
end
```

```irb
irb> Person.new.valid?
ActiveModel::StrictValidationFailed: Name can’t be blank
```

Также возможно передать собственное исключение в опцию `:strict`.

```ruby
class Person < ApplicationRecord
  validates :token, presence: true, uniqueness: true, strict: TokenGenerationException
end
```

```irb
irb> Person.new.valid?
TokenGenerationException: Token can’t be blank
```

(conditional-validation) Условная валидация
-------------------------------------------

Иногда имеет смысл проводить валидацию объекта только при выполнении заданного предиката. Это можно сделать, используя опции `:if` и `:unless`, которые принимают символ, `Proc` или `Array`. Опцию `:if` можно использовать, если необходимо определить, когда валидация **должна** произойти. Альтернативно, если нужно определить, когда валидация **не должна** произойти, воспользуйтесь опцией `:unless`.

### Использование символа с `:if` и `:unless`

Вы можете связать опции `:if` и `:unless` с символом, соответствующим имени метода, который будет вызван перед валидацией. Это наиболее часто используемый вариант.

```ruby
class Order < ApplicationRecord
  validates :card_number, presence: true, if: :paid_with_card?

  def paid_with_card?
    payment_type == "card"
  end
end
```

### Использование Proc с `:if` и `:unless`

Можно связать `:if` и `:unless` с объектом `Proc`, который будет вызван. Использование объекта `Proc` дает возможность написать встроенное условие вместо отдельного метода. Этот вариант лучше всего подходит для однострочного кода.

```ruby
class Account < ApplicationRecord
  validates :password, confirmation: true,
    unless: Proc.new { |a| a.password.blank? }
end
```

Так как `lambda` это тип `Proc`, она также может быть использована для написания вложенных условий, используя преимущество сокращенного синтаксиса.

```ruby
validates :password, confirmation: true, unless: -> { password.blank? }
```

### Группировка условных валидаций

Иногда полезно иметь несколько валидаций с одним условием. Это легко достигается с использованием [`with_options`][].

```ruby
class User < ApplicationRecord
  with_options if: :is_admin? do |admin|
    admin.validates :password, length: { minimum: 10 }
    admin.validates :email, presence: true
  end
end
```

Во все валидации внутри `with_options` будет автоматически передано условие `if: :is_admin?`.

[`with_options`]: https://api.rubyonrails.org/classes/Object.html#method-i-with_options

### Объединение условий валидации

С другой стороны, может использоваться массив, когда несколько условий определяют, должна ли произойти валидация. Более того, в одной и той же валидации можно применить и `:if:`, и `:unless`.

```ruby
class Computer < ApplicationRecord
  validates :mouse, presence: true,
                    if: [Proc.new { |c| c.market.retail? }, :desktop?],
                    unless: Proc.new { |c| c.trackpad.present? }
end
```

Валидация выполнится только тогда, когда все условия `:if` и ни одно из условий `:unless` будут вычислены со значением `true`.

(performing-custom-validations) Выполнение собственных валидаций
---------------------------------

Когда встроенных валидационных хелперов недостаточно для ваших нужд, можете написать свои собственные валидаторы или методы валидации.

### Собственные валидаторы

Собственные валидаторы это классы, наследуемые от [`ActiveModel::Validator`][]. Эти классы должны реализовать метод `validate`, принимающий запись как аргумент и выполняющий валидацию на ней. Собственный валидатор вызывается с использованием метода `validates_with`.

```ruby
class MyValidator < ActiveModel::Validator
  def validate(record)
    unless record.name.start_with? 'X'
      record.errors.add :name, "Need a name starting with X please!"
    end
  end
end
 
class Person < ApplicationRecord
  validates_with MyValidator
end
```

Простейшим способом добавить собственные валидаторы для валидации отдельных атрибутов является наследуемость от [`ActiveModel::EachValidator`][]. В этом случае класс собственного валидатора должен реализовать метод `validate_each`, принимающий три аргумента: запись, атрибут и значение. Это будут соответствующие экземпляр, атрибут, который будет проверяться и значение атрибута в переданном экземпляре:

```ruby
class EmailValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    unless URI::MailTo::EMAIL_REGEXP.match?(value)
      record.errors.add attribute, (options[:message] || "is not an email")
    end
  end
end

class Person < ApplicationRecord
  validates :email, presence: true, email: true
end
```

Как показано в примере, можно объединять стандартные валидации со своими произвольными валидаторами.

[`ActiveModel::EachValidator`]: https://api.rubyonrails.org/classes/ActiveModel/EachValidator.html
[`ActiveModel::Validator`]: https://api.rubyonrails.org/classes/ActiveModel/Validator.html

### Собственные методы

Также возможно создать методы, проверяющие состояние ваших моделей и добавляющие ошибки в коллекцию `errors`, когда они невалидны. Затем эти методы следует зарегистрировать, используя метод класса [`validate`][], передав символьные имена валидационных методов.

Можно передать более одного символа для каждого метода класса, и соответствующие валидации будут запущены в том порядке, в котором они зарегистрированы.

Метод `valid?` проверит, что коллекция `errors` пуста, поэтому ваши собственные методы валидации должны добавить ошибки в нее, когда вы хотите, чтобы валидация провалилась:

```ruby
class Invoice < ApplicationRecord
  validate :expiration_date_cannot_be_in_the_past,
    :discount_cannot_be_greater_than_total_value

  def expiration_date_cannot_be_in_the_past
    if expiration_date.present? && expiration_date < Date.today
      errors.add(:expiration_date, "can't be in the past")
    end
  end

  def discount_cannot_be_greater_than_total_value
    errors.add(:discount, "can't be greater than total value") if
      discount > total_value
  end
end
```

По умолчанию такие валидации будут выполнены каждый раз при вызове `valid?` или сохранении объекта. Но также возможно контролировать, когда выполнять собственные валидации, передав опцию `:on` в метод `validate`, с ключами: `:create` или `:update`.

```ruby
class Invoice < ApplicationRecord
  validate :active_customer, on: :create

  def active_customer
    errors.add(:customer_id, "is not active") unless customer.active?
  end
end
```

Подробности об [`:on`](#on) смотрите в разделе выше.

### Список валидаторов

Если хотите обнаружить все валидаторы для данного объекта, не ищите что-то другое, кроме `validators`.

Например, если у нас есть следующая модель, использующая пользовательский валидатор и встроенный валидатор:

```ruby
class Person < ApplicationRecord
  validates :name, presence: true, on: :create
  validates :email, format: URI::MailTo::EMAIL_REGEXP
  validates_with MyOtherValidator, strict: true
end
```

Теперь можно использовать `validators` на модели "Person" для перечисления всех валидаторов, или даже проверяющих определенного поля с помощью `validators_on`.

```irb
irb> Person.validators
#=> [#<ActiveRecord::Validations::PresenceValidator:0x10b2f2158
      @attributes=[:name], @options={:on=>:create}>,
     #<MyOtherValidatorValidator:0x10b2f17d0
      @attributes=[:name], @options={:strict=>true}>,
     #<ActiveModel::Validations::FormatValidator:0x10b2f0f10
      @attributes=[:email],
      @options={:with=>URI::MailTo::EMAIL_REGEXP}>]
     #<MyOtherValidator:0x10b2f0948 @options={:strict=>true}>]

irb> Person.validators_on(:name)
#=> [#<ActiveModel::Validations::PresenceValidator:0x10b2f2158
      @attributes=[:name], @options={on: :create}>]
```

[`validate`]: https://api.rubyonrails.org/classes/ActiveModel/Validations/ClassMethods.html#method-i-validate

(working-with-validation-errors) Работаем с ошибками валидации
--------------------------------------------------------------

Методы [`valid?`][] и [`invalid?`][] предоставляют только итоговый статус валидности. Однако, можно погрузиться в каждую отдельную ошибку с помощью различных методов из коллекции `errors`.

Предлагаем список наиболее часто используемых методов. Если хотите увидеть список всех доступных методов, обратитесь к документации по [`ActiveModel::Errors`][].

[`ActiveModel::Errors`]: https://api.rubyonrails.org/classes/ActiveModel/Errors.html

### `errors`

Шлюз, через который можно извлечь различные подробности о каждой ошибке.

Он возвращает экземпляр класса `ActiveModel::Errors`, содержащий все ошибки, каждая ошибка представлена объектом [`ActiveModel::Error`][].

```ruby
class Person < ApplicationRecord
  validates :name, presence: true, length: { minimum: 3 }
end
```

```irb
irb> person = Person.new
irb> person.valid?
=> false
irb> person.errors.full_messages
=> ["Name can’t be blank", "Name is too short (minimum is 3 characters)"]

irb> person = Person.new(name: "John Doe")
irb> person.valid?
=> true
irb> person.errors.full_messages
=> []

irb> person = Person.new
irb> person.valid?
=> false
irb> person.errors.first.details
=> {:error=>:too_short, :count=>3}
```

[`ActiveModel::Error`]: https://api.rubyonrails.org/classes/ActiveModel/Error.html

### `errors[]`

[`errors[]`][Errors#squarebrackets] используется, когда вы хотите проверить сообщения об ошибке для определенного атрибута. Он возвращает массив строк со всеми сообщениями об ошибке для заданного атрибута, каждая строка с одним сообщением об ошибке. Если нет ошибок, относящихся к атрибуту, возвратится пустой массив.

```ruby
class Person < ApplicationRecord
  validates :name, presence: true, length: { minimum: 3 }
end
```

```irb
irb> person = Person.new(name: "John Doe")
irb> person.valid?
=> true
irb> person.errors[:name]
=> []

irb> person = Person.new(name: "JD")
irb> person.valid?
=> false
irb> person.errors[:name]
=> ["is too short (minimum is 3 characters)"]

irb> person = Person.new
irb> person.valid?
=> false
irb> person.errors[:name]
=> ["can’t be blank", "is too short (minimum is 3 characters)"]
```

### `errors.where` и объект ошибки

Иногда нам нужно больше подробностей о каждой ошибке, кроме ее сообщения. Каждая ошибка инкапсулирована как объект `ActiveModel::Error`, и метод [`where`][] является наиболее простым способом доступа.

`where` возвращает массив объектов ошибки, фильтрованный различными условиями.

```ruby
class Person < ApplicationRecord
  validates :name, presence: true, length: { minimum: 3 }
end
```

Можно отфильтровать только по `attribute`, передав его как первый параметр в `errors.where(:attr)`. Второй параметр используется для фильтрации по `type` требуемой ошибки, вызывая `errors.where(:attr, :type)`.

```irb
irb> person = Person.new
irb> person.valid?
=> false

irb> person.errors.where(:name)
=> [ ... ] # все ошибки, связанные с атрибутом :name

irb> person.errors.where(:name, :too_short)
=> [ ... ] # ошибки :too_short для атрибута :name
```

Наконец, можно отфильтровать по любой `options`, существующих для данного типа объекта ошибки.

```irb
irb> person = Person.new
irb> person.valid?
=> false

irb> person.errors.where(:name, :too_short, minimum: 3)
=> [ ... ] # all name errors being too short and minimum is 2
```

Из этих объектов ошибки можно считывать разную информацию:

```irb
irb> error = person.errors.where(:name).last

irb> error.attribute
=> :name
irb> error.type
=> :too_short
irb> error.options[:count]
=> 3
```

Также можно сгенерировать сообщение об ошибке:

```irb
irb> error.message
=> "is too short (minimum is 3 characters)"
irb> error.full_message
=> "Name is too short (minimum is 3 characters)"
```

Метод [`full_message`][] генерирует более дружелюбное сообщение с добавлением имени атрибута с большой буквы вначале. (Чтобы настроить формат, используемый `full_message`, посмотрите [руководство I18n](/i18n#active-model-methods).)

[`full_message`]: https://api.rubyonrails.org/classes/ActiveModel/Errors.html#method-i-full_message
[`where`]: https://api.rubyonrails.org/classes/ActiveModel/Errors.html#method-i-where

### `errors.add`

Метод [`add`][] создает объект ошибки, принимая `attribute`, `type` ошибки и дополнительный хэш опций. Это полезно при написания собственного валидатора, так как это позволяет определить очень специфичные ситуации с ошибками.

```ruby
class Person < ApplicationRecord
  validate do |person|
    errors.add :name, :too_plain, message: "is not cool enough"
  end
end
```

```irb
irb> person = Person.create
irb> person.errors.where(:name).first.type
=> :too_plain
irb> person.errors.where(:name).first.full_message
=> "Name is not cool enough"
```

[`add`]: https://api.rubyonrails.org/classes/ActiveModel/Errors.html#method-i-add

### `errors[:base]`

Можете добавлять ошибки, которые относятся к состоянию объекта в целом, а не к отдельному атрибуту. Для этого необходимо использовать `:base` как атрибут при добавлении новой ошибки.

```ruby
class Person < ApplicationRecord
  validate do |person|
    errors.add :base, :invalid, message: "This person is invalid because ..."
  end
end
```

```irb
irb> person = Person.create
irb> person.errors.where(:base).first.full_message
=> "This person is invalid because ..."
```

### `errors.size`

Метод `size` возвращает количество ошибок для объекта.

```ruby
class Person < ApplicationRecord
  validates :name, presence: true, length: { minimum: 3 }
end
```

```irb
irb> person = Person.new
irb> person.valid?
=> false
irb> person.errors.size
=> 2

irb> person = Person.new(name: "Andrea", email: "andrea@example.com")
irb> person.valid?
=> true
irb> person.errors.size
=> 0
```

### `errors.clear`

Метод `clear` используется, когда вы намеренно хотите очистить коллекцию `errors`. Естественно, вызов `errors.clear` для невалидного объекта фактически не сделает его валидным: сейчас коллекция `errors` будет пуста, но в следующий раз, когда вы вызовете `valid?` или любой метод, который пытается сохранить этот объект в базу данных, валидации выполнятся снова. Если любая из валидаций провалится, коллекция `errors` будет заполнена снова.

```ruby
class Person < ApplicationRecord
  validates :name, presence: true, length: { minimum: 3 }
end
```

```irb
irb> person = Person.new
irb> person.valid?
=> false
irb> person.errors.empty?
=> false

irb> person.errors.clear
irb> person.errors.empty?
=> true

irb> person.save
=> false

irb> person.errors.empty?
=> false
```

(displaying-validation-errors-in-the-view) Отображение ошибок валидации во вью
---------------------------------------------------------------------------------

Как только вы создали модель и добавили валидации, если эта модель создается с помощью веб-формы, то вы, возможно хотите отображать сообщение об ошибке, когда одна из валидаций проваливается.

Поскольку каждое приложение обрабатывает подобные вещи по-разному, в Rails нет какого-то хелпера вью для непосредственной генерации этих сообщений. Однако, благодаря богатому набору методов, которые Rails в целом дает для взаимодействия с валидациями, можно создать свой собственный. Кроме того, при генерации скаффолда, Rails поместит некоторый ERB в `_form.html.erb`, генерируемый для отображения полного списка ошибок этой модели.

Допустим, у нас имеется модель, сохраненная в переменную экземпляра `@article`, это выглядит следующим образом:

```html+erb
<% if @article.errors.any? %>
  <div id="error_explanation">
    <h2><%= pluralize(@article.errors.count, "error") %> prohibited this article from being saved:</h2>
    <ul>
      <% @article.errors.each do |error| %>
        <li><%= error.full_message %></li>
      <% end %>
    </ul>
  </div>
<% end %>
```

Более того, при использовании хелперов форм Rails для создания форм, когда у поля происходит ошибка валидации, генерируется дополнительный `<div>` вокруг содержимого.

```html
<div class="field_with_errors">
  <input id="article_title" name="article[title]" size="30" type="text" value="">
</div>
```

Этот div можно стилизовать по желанию. К примеру, дефолтный скаффолд, который генерирует Rails, добавляет это правило CSS:

```css
.field_with_errors {
  padding: 2px;
  background-color: red;
  display: table;
}
```

Это означает, что любое поле с ошибкой обведено красной рамкой толщиной 2 пикселя.
