---
layout: post
title: 'Сервисы: Rails и Symfony'
date: 2018-01-12 12:14 +0500
permalink: /services-rails-and-symfony/
---
Долгое время муссировалась и&nbsp;продолжает муссироваться в&nbsp;кругах разных каркасов идея сервисов. Во&nbsp;многих фреймворках сервисы стали самостоятельными единицами кода (как скажем, в&nbsp;Symfony о&nbsp;чем мы&nbsp;еще поговорим). В&nbsp;Rails&nbsp;же паттерн Service Object выглядит очень просто, так как никаких &laquo;самостоятельных единиц&raquo; именуемых сервисами, в&nbsp;нем нет.

В&nbsp;этой статье я&nbsp;покажу сначала разработку страницы, использующей сервис в&nbsp;Symfony&nbsp;3.3.9&nbsp;а затем&nbsp;&mdash; то&nbsp;же самое в&nbsp;Rails. Сразу скажу читатели раздела Rails будут слегка разочарованы&nbsp;&mdash; &laquo;он&nbsp;опять одну строчку написал&raquo;. Ну&nbsp;извините, это&nbsp;же пример. 

Несмотря на&nbsp;то&nbsp;что я&nbsp;являюсь ROR разработчиком, каркас Symfony мне очень понравился. Я&nbsp;считаю его документацию и&nbsp;продукты такие как twig&nbsp;&mdash; лучшими в&nbsp;нише. Надеюсь, с&nbsp;выходом версии 4&nbsp;такая тенденция не&nbsp;утеряется. А&nbsp;выход этот уже произошел. 

Итак, что будет за&nbsp;задача? 

Почешем там где у&nbsp;нас чешется&nbsp;&mdash; есть у&nbsp;меня просто папка в&nbsp;локальной файловой системе, там находится куча шахматных книжек. Я&nbsp;хочу сделать страницу где&nbsp;бы они перечислялись. 

Для этого мы&nbsp;и&nbsp;используем сервисы&nbsp;&mdash; в&nbsp;нашем случае сервис какой возвращает список книг (файлов) в&nbsp;удобном для нас виде. Минуя директории.

### Делаем это на Symfony

Программисты на&nbsp;чистом PHP сразу&nbsp;же бросятся делать readdir и&nbsp;много чего другого в&nbsp;стиле &laquo;да&nbsp;я&nbsp;щас за&nbsp;пять минут&raquo;. Мы&nbsp;тоже не&nbsp;будем усложнять, но&nbsp;использовать мы&nbsp;будем нормальный подход&nbsp;&mdash; генератор [DirectoryIterator][directory-iterator].

Итак, у&nbsp;нас есть Symfony&nbsp;3.3.9. Создадим-ка контроллер книг, какой собственно будет запускаться для отображения страницы

{% highlight php %}
<?php
// src/AppBundle/Controller/BooksController.php

namespace AppBundle\Controller;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use AppBundle\Service\DirectoryLister;

class BooksController extends AbstractController
{
    /**
     * @Route("/books/list", name="books_list")
     */
    public function listAction(DirectoryLister $dirService)
    {
        $files = $dirService->getFileList();
        return $this->render('books/list.html.twig', ['files' => $files]);
    }
}
{% endhighlight %}

Как видим, контроллер у&nbsp;нас простой. Он&nbsp;обращается к&nbsp;сервису, какой внедряется к&nbsp;нему в&nbsp;качестве зависимости с&nbsp;помощью&nbsp;DI, получает список файлов. А&nbsp;потом просто отображает шаблон, передав туда этот список.

Обратите внимание, что класс контроллера наследован от&nbsp;AbstractController. Это налагает ограничение на&nbsp;вызовы сервисов из&nbsp;сервис-контейнера внутри экшена, и&nbsp;делает разработку более строгой. Теперь, наследовав так, мы&nbsp;можем только совершать инжект зависимости (при помощи type-hint) но&nbsp;не&nbsp;вызывать эту зависимость из&nbsp;сервис-контейнера внутри экшена. 

Эти действия предназначены для более строгой разработки, чтобы не&nbsp;было случаев когда у&nbsp;нас есть экшн какой делает 10&nbsp;дел и&nbsp;распухает в&nbsp;&laquo;метод-Бог&raquo; а&nbsp;мы&nbsp;этого или не&nbsp;видим, или не&nbsp;желаем видеть. В&nbsp;случае&nbsp;же инжектов сразу ясно&nbsp;&mdash; если у&nbsp;вас параметры метода содержат 20&nbsp;инжектов значит происходит что-то очень нехорошее и&nbsp;вам нужно срочно рефакторить приложение.

Займемся сочинением непосредственно нашего сервиса:

{% highlight php %}
<?php
// src/AppBundle/Service/DirectoryLister.php

namespace AppBundle\Service;
use DirectoryIterator;
class DirectoryLister
{
    private $path;
    public function __construct($path)
    {
        $this->path = $path;
    }
    public function getFileList()
    {
        $directoryIteratorInstance = new DirectoryIterator($this->path);
        foreach ($directoryIteratorInstance as $fileNode) {
            if (!$fileNode->isDir()) {
                $files[] = $fileNode->getFilename();
            }
        }
        sort($files);
        return $files;
    }
}
{% endhighlight %}

Сервис наш должен быть многоцелевым, применяться неоднократно, повторно использоваться. По&nbsp;этому в&nbsp;него с&nbsp;помощью аргумента конструктора внедряется параметр path какой указывает ИЗ&nbsp;КАКОЙ директории собственно, возвращать список файлов. В&nbsp;Symfony это называется Autowiring. Так&nbsp;же отметьте, что мы&nbsp;используем генератор [DirectoryIterator][directory-iterator], а&nbsp;не&nbsp;наколенный перебор файлов в&nbsp;директории. Это дает нам удобные методы isDir и&nbsp;другие.

Куда&nbsp;же теперь поместить параметр пути к&nbsp;файлам? Симфони дает однозначный ответ, это файл

{% highlight bash %}
app/config/parameters.yml
{% endhighlight %}

Он&nbsp;по&nbsp;умолчанию не&nbsp;ставится на&nbsp;контроль версий git, и&nbsp;вообще предназначен для &laquo;переносных&raquo; меняющихся настроек. Вот туда-то мы&nbsp;и&nbsp;положим значение для сервиса, какое он&nbsp;автоматически прочитает:

{% highlight yml %}
parameters:
    #other params
    books_path: /home/izotoff/Документы/Шахматы
{% endhighlight %}

Но&nbsp;это еще не&nbsp;все. В&nbsp;файле

{% highlight bash %}
app/config/Services.yml
{% endhighlight %}

нам необходимо указать, что наш сервис принимает этот параметр:

{% highlight yml %}
AppBundle\Service\DirectoryLister:
    arguments:
        $path: "%books_path%"
{% endhighlight %}   

Отлично! Все почти готово. Если сейчас обратиться по&nbsp;адресу /app_dev.php/books/list вашего хоста, выскочит ошибка о&nbsp;том что не&nbsp;найден шаблон twig. Поправим это:

{% gist d34a929d5a255d475d20f83b5e2986db %}

Все теперь работает. Полюбуемся на&nbsp;результат, зайдя на&nbsp;хост (у&nbsp;меня это symfony.local) то&nbsp;есть мой адрес будет http://symfony.local/app_dev.php/books/list а&nbsp;продакшн http://symfony.local/books/list. Не&nbsp;забудьте перед запуском продакшна сделать регенерацию кеша симфони (в&nbsp;том числе он&nbsp;кеширует роуты):

{% highlight bash %}
bin/console cache:clear --env=prod
{% endhighlight %} 

Сделав все эти манипуляции с&nbsp;кешем, или зайдя как на&nbsp;скрине&nbsp;&mdash; через app_dev.php то&nbsp;есть с&nbsp;поддержкой дебаг тулбара, мы&nbsp;увидим следующую милую картину: ([посмотреть увеличенную][symfony-screen]):

![Список книг локальной ФС с сервисом на Symfony](https://habrastorage.org/webt/fp/m8/pa/fpm8pas0bsrcqosmvzvoomvtryo.png){:class="img-responsive"}

### Делаем это же на Ruby on Rails

Уфф, тут кода будет меньше, и&nbsp;телодвижений тоже. Но&nbsp;это не&nbsp;значит что Symfony чем-то хуже или что он&nbsp;громоздок. Концепция сервисов как практикуемых единиц очень полезна и&nbsp;хороша. В&nbsp;Rails&nbsp;же мы&nbsp;начнем с&nbsp;простого. Сначала создадим само приложение:

{% highlight bash %}
rails new booksdemo && cd booksdemo
{% endhighlight %} 

А&nbsp;теперь создадим-ка контроллер, для простоты с&nbsp;всего одним действием, как и&nbsp;в&nbsp;Symfony:

{% highlight bash %}
rails g controller Books index
{% endhighlight %} 

Чтобы использовать паттерн Service Object, по&nbsp;сути в&nbsp;Rails достаточно положить файл представляющий &laquo;сервис&raquo; куда-нибудь в&nbsp;такое место, где он&nbsp;будет автоматически загружен. Создадим файл-сервис в&nbsp;app/services, по&nbsp;умолчанию этой папки нет но&nbsp;сейчас появится:

{% highlight bash %}
mkdir app/services && touch app/services/directory_list_service.rb
{% endhighlight %} 

Теперь у&nbsp;нас есть файл, какой будет загружен Rails самостоятельно. Заполним его:

{% highlight ruby %}
#app/services/directory_list_service.rb
class DirectoryListService
  attr_reader :files

  def initialize(path)
    @files = Dir.glob(path).select { |filename| File.file?(filename) }.map { |fullpath| File.basename fullpath }.sort
  end


end
{% endhighlight %} 

Вот и&nbsp;весь сервис. За&nbsp;счет гибкости Ruby мы&nbsp;получили гораздо меньше кода для перебора директории.

Используем его в&nbsp;контроллере:

{% highlight ruby %}
#app/controllers/books_controller.rb
class BooksController < ApplicationController
  def index
    @files = DirectoryListService.new('/home/izotoff/Документы/Шахматы/*').files
  end
end
{% endhighlight %}

И&nbsp;наконец, заполним вид (view):

{% highlight erb %}
<h1>Книги (<%= @files.count %>)</h1>
<% @files.each do |file| %>
	<div><%= file %></div>
<% end %>
{% endhighlight %}

Вот и&nbsp;все! Теперь мы&nbsp;увидим мало отличающуюся от&nbsp;Symfony-реализации картину, запустив

{% highlight bash %}
rails s
{% endhighlight %} 

и перейдя на http://localhost:3000/books/index :

![Список книг локальной ФС с сервисом на Ruby on Rails](https://habrastorage.org/webt/fl/kk/1i/flkk1ir-3vlqg1e6a6vxiphhcws.png){:class="img-responsive"}

Тоже [можно увеличить][rails-screen].

Теперь поговорим о&nbsp;гибкости решения. Во-первых поскольку мы&nbsp;в&nbsp;сервисе используем Dir.glob то&nbsp;мы&nbsp;ставим звездочку в&nbsp;конце нашего пути к&nbsp;файлам. Во-вторых этот самый путь мы&nbsp;прописываем прямо там, где сервис используется&nbsp;&mdash; в&nbsp;контроллере. Это не&nbsp;совсем гибко! Но&nbsp;идеология Rails состоит как раз в&nbsp;том чтобы избавляться от&nbsp;внешних файлов настроек, даже таких безобидных как parameters.yml в&nbsp;Symfony. Да, мы&nbsp;конечно можем все-таки оформить путь к&nbsp;файлам как статичную настройку. Но&nbsp;делать это по&nbsp;крайней мере, в&nbsp;данной статье мы&nbsp;не&nbsp;будем. Не&nbsp;потому что &laquo;и&nbsp;так сойдет&raquo; а&nbsp;потому что это Rails. И&nbsp;не&nbsp;факт что другой контроллер запросит уже иной путь, тогда наше решение выиграет по&nbsp;гибкости так как путь можно указывать создавая класс объекта-сервиса.

Подведя итоги, скажу что оба каркаса хоть я&nbsp;и&nbsp;за&nbsp;ROR, прекрасно справляются с&nbsp;задачей, просто каждый своим путем. В&nbsp;Symfony можно тоже сделать сервис гибким, чтобы передавать в&nbsp;него параметр при вызове каждый раз но&nbsp;это я&nbsp;оставил &laquo;за&nbsp;кулисами&raquo; статьи, предпочтя автосвязывание аргументов из&nbsp;файла настроек.

[directory-iterator]: http://php.net/manual/ru/class.directoryiterator.php
[symfony-screen]: https://habrastorage.org/webt/fp/m8/pa/fpm8pas0bsrcqosmvzvoomvtryo.png
[rails-screen]: https://habrastorage.org/webt/fl/kk/1i/flkk1ir-3vlqg1e6a6vxiphhcws.png