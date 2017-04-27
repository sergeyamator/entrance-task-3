# Задание 3

Мобилизация.Гифки – сервис для поиска гифок в перерывах между занятиями.

Сервис написан с использованием [bem-components](https://ru.bem.info/platform/libs/bem-components/5.0.0/).

Работа избранного в оффлайне реализована с помощью технологии [Service Worker](https://developer.mozilla.org/ru/docs/Web/API/Service_Worker_API/Using_Service_Workers).

Для поиска изображений используется [API сервиса Giphy](https://github.com/Giphy/GiphyAPI).

В браузерах, не поддерживающих сервис-воркеры, приложение так же должно корректно работать, 
за исключением возможности работы в оффлайне.

## Структура проекта

  * `gifs.html` – точка входа
  * `assets` – статические файлы проекта
  * `vendor` –  статические файлы внешних библиотек
  * `service-worker.js` – скрипт сервис-воркера

Открывать `gifs.html` нужно с помощью локального веб-сервера – не как файл. 
Это можно сделать с помощью встроенного в WebStorm/Idea веб-сервера, с помощью простого сервера
из состава PHP или Python. Можно воспользоваться и любым другим способом.

## Действия которые делал для решения проблемы:

  * До этого никогда не работал с данной технологией поэтому сразу пошел на mdn читать руководство 
  * Открыл инструмет разработчика, увидел что ошибок нет
    и сообщение blocks.js:471 [ServiceWorkerContainer] ServiceWorker is registered!
  * Открыл файле blocks.js и начал искать регистрацию воркера что бы быть уверен что он вообще регистрируется,
    в целом я его нашел на 463 строке 
  * Открыл файл service-worker.js начал просматривать его, заметил что есть вопросы, на которые
    я начал отвечать. 
  * Перенес файл service-worker.js в корень и на 469 строке добавил опцию scope: './', так как увидел данную
    опцию в одном из видеоуроков на youtube канале
  * После этого начал проверять в офлайн режиме и понял что у нас кешируются не все файлы, так
    как в функции needStoreForOffline они не указаны, не было html, webp картинок, а так же
    запроса на api.giphy.com я их добавил. Как я понял - открыл консоль и там было красным выделены те файла, 
    которые не грузятся.
  * В итоге странички успешно отображались в оффлайн режиме, но к сожалению и в он-лайне
  * После этого пытался искать в чем проблема но к сожалению не смог. В итоге было принято решение открыть
    чужие форки и посмотреть кто как и что делал, наткнулся на решение - нужно удалить 
    caches.match(cacheKey).then(cacheResponse => cacheResponse на 46 строке в методе fetch.
  * А так же добавить ресурсы, которые будут кешироваться сразу после первого запроса в методе install
     
## Ответы на вопросы
   * Вопрос №1: зачем нужен этот вызов?
   
   > .then(() => preCacheAllFiles())
     .then(() => self.skipWaiting())
   
   В данном обработчике обычно  кешируются необходимые файлы что
   здесь и происходит, мы вызываем функцию preCacheAllFavorites которая
   ложит в новый кеш все добавленные в избранное картинки
   skipWaiting - Делает активный сервис воркер который находился в стадии ожидания.
     
   * Вопрос №2: зачем нужен этот вызов? 
   >  deleteObsoleteCaches()
             .then(() => { 
              self.clients.claim(); 
                
   Данный вызов удаляет неактуальный кеш. Это очень хорошо что происходит
   в данном обработчике события, так как он идеально подходит для этого.
   Если же удалитить какой-либо из старых кешей в обработчике install,
   старый Service Worker, который в настоящий момент контролирует страницу,
   не сможет получить к нему доступ.
                                
   Метод claim() интерфейса Clients позволяет активному сервис воркеру установить себя
   как активного воркера для клиентской страницы, когда воркер и страница находятся в одной области.
   Он запускает событие oncontrollerchange на всех клиентских страницах в пределах области сервис воркера.

   claim() внутри обработчика события onActivate сервис воркера, так что клиентская страница,
   загруженая в той же области, не нуждается в перезагрузке прежде чем она может быть использована
   сервис воркером.  
                              
   * Вопрос №3: для всех ли случаев подойдёт такое построение ключа?  
   > const cacheKey = url.origin + url.pathname;
                            
  Если честно затрудняюсь ответить.
  Конкретно в нашем случае мы получаем такие [url](http://joxi.ru/a2XakDLh1J7WQA)
  Что для нашего случая нормально.                    
       
  * Вопрос №4: зачем нужна эта цепочка вызовов?
  >     return Promise.all(
  >     names.filter(name => name !== CACHE_VERSION)
  >          .map(name => {
  >              console.log('[ServiceWorker] Deleting obsolete cache:', name);
  >              return caches.delete(name);
  >          })
  >     );
  
  При регистрации мы задали определенную версию кешей.
  Теперь мы проверяем соответствуют ли закешированные ресурсы этой версии и отфильтровуем их, если
  нет тогда удаляем.
  Promise all используется так как names это массив версий и нам нужно пройтись по ним всем,
  и удалить кеша, а caches.delete(name) это асинхронний метод удаления кеша по ключу (в нашем
  случае это версия кеша ), поэтому и promise
  
  * Нужно ли при скачивании сохранять ресурс для оффлайна?
  > function needStoreForOffline(cacheKey) {
      return cacheKey.includes('vendor/') ||
    
  Да, нужно. Так как при оффлайне нам нужны все эти файлы
  для корректной работы сайта
  
  * Вопрос №5: для чего нужно клонирование?
  > cache.put(cacheKey, response.clone());   
                     
 Так как это stream, мы должны клонировать для
 того что б иметь возможность использовать много раз.

 Потому, что потоки запроса и ответа могут быть прочитаны только единожды

 Чтобы ответ был получен браузером и сохранен в кеше — нам нужно клонировать его.
 Так, оригинальный объект отправится браузеру, а клон будет закеширован.
 Оба они будут прочитаны единожды.
                         
   
    