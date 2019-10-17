
## word2vec

Мы уже работали с грамматикой — в автоматическом кодировании грамматических категорий, кажется, ничего сверхъестественного нет. Но можно ли как-то закодировать значение слова? Да! В этом нам поможет лингвистическая теория, которая называется дистрибутивной гипотезой. Она утверждает, что значение слова определяется его контекстом — иначе говоря, словами, которые встречаются рядом с этим словом в тексте. Область лингвистики, которая занимается вычислением степени семантической близости между словами/текстами и т.п. на основании их распределения (дистрибуции) в больших массивах данных (текстовых корпусах) назвается **дистрибутивной семантикой**.

Одной из самых известных моделей для работы с дистрибутивной семантикой является word2vec. Технология основана на нейронной сети, предсказывающей вероятность встретить слово в заданном контексте. Этот инструмент был разработан группой исследователей Google в 2013 году, руководителем проекта был Томаш Миколов (сейчас работает в Facebook). Вот две самые главные статьи:

+ [Efficient Estimation of Word Representations inVector Space](https://arxiv.org/pdf/1301.3781.pdf)
+ [Distributed Representations of Words and Phrases and their Compositionality](https://arxiv.org/abs/1310.4546)

Полученные таким образом вектора называются распределенными представлениями слов, или **эмбеддингами**.

#### Как это обучается?

Мы задаём вектор для каждого слова с помощью матрицы $w$ и вектор контекста с помощью матрицы $W$. По сути, word2vec является обобщающим названием для двух архитектур Skip-Gram и Continuous Bag-Of-Words (CBOW).

+ **CBOW** предсказывает текущее слово, исходя из окружающего его контекста.

+ **Skip-gram**, наоборот, использует текущее слово, чтобы предугадывать окружающие его слова.

#### Как это работает?

Word2vec принимает большой текстовый корпус в качестве входных данных и сопоставляет каждому слову вектор, выдавая координаты слов на выходе. Сначала он создает словарь, «обучаясь» на входных текстовых данных, а затем вычисляет векторное представление слов. Векторное представление основывается на контекстной близости: слова, встречающиеся в тексте рядом с одинаковыми словами (а следовательно, согласно дистрибутивной гипотезе, имеющие схожий смысл), в векторном представлении будут иметь близкие координаты векторов-слов. Для вычисления близости слов используется косинусное расстояние между их векторами.

С помощью дистрибутивных векторных моделей можно строить семантические пропорции (они же аналогии) и решать примеры:

+ король: мужчина = королева: женщина $\Rightarrow$
+ король - мужчина + женщина = королева

![w2v](https://cdn-images-1.medium.com/max/2600/1*sXNXYfAqfLUeiDXPCo130w.png)

Ещё про механику с картинками [тут](https://habr.com/ru/post/446530/)

#### Зачем это нужно?

+ используется для решения семантических задач
+ давайте подумаем, для описания каких семантических классов слов дистрибутивная информация особенно важна?
+ несколько интересных статей по дистрибутивной семантике:

* [Turney and Pantel 2010](https://jair.org/index.php/jair/article/view/10640)
* [Lenci 2018](https://www.annualreviews.org/doi/abs/10.1146/annurev-linguistics-030514-125254?journalCode=linguistics)
* [Smith 2019](https://arxiv.org/pdf/1902.06006.pdf)
* [Pennington et al. 2014](https://www.aclweb.org/anthology/D14-1162/)
* [Faruqui et al. 2015](https://www.aclweb.org/anthology/N15-1184/)

+ подаётся на вход нейронным сетям
+ используется в Siri, Google Assistant, Alexa, Google Translate...

#### Gensim

Использовать предобученную модель эмбеддингов или обучить свою можно с помощью библиотеки `gensim`. Вот ее [документация](https://radimrehurek.com/gensim/models/word2vec.html). Вообще-то `gensim` — библиотека для тематического моделирования текстов, но один из компонентов в ней — реализация на python алгоритмов из библиотеки word2vec (которая в оригинале была написана на C++).

Если gensim у вас не стоит, то ставим: `pip install gensim`. Можно сделать это прямо из jupyter'а! Чтобы выполнить какую-то команду не в питоне, в командной строке, нужно написать перед ней восклицательный знак.



```python
import re
import gensim
import logging
import nltk.data
import pandas as pd
import urllib.request
from bs4 import BeautifulSoup
from nltk.corpus import stopwords
from gensim.models import word2vec
```

    C:\ProgramData\Anaconda3\lib\site-packages\smart_open\ssh.py:34: UserWarning: paramiko missing, opening SSH/SCP/SFTP paths will be disabled.  `pip install paramiko` to suppress
      warnings.warn('paramiko missing, opening SSH/SCP/SFTP paths will be disabled.  `pip install paramiko` to suppress')
    

#### Как обучить свою модель

NB! Обратите внимание, что тренировка модели не включает препроцессинг! Это значит, что избавляться от пунктуации, приводить слова к нижнему регистру, лемматизировать их, проставлять частеречные теги придется до тренировки модели (если, конечно, это необходимо для вашей задачи). Т.е. в каком виде слова будут в исходном тексте, в таком они будут и в модели.

Поскольку иногда тренировка модели занимает много времени, то можно ещё вести лог событий, чтобы понимать, что на каком этапе происходит.


```python
logging.basicConfig(format='%(asctime)s : %(levelname)s : %(message)s', level=logging.INFO)
```


```python
# ЭТУ ЯЧЕЙКУ ЗАПУСКАТЬ НЕ НАДО

from tqdm import tqdm
from pymystem3 import Mystem

m = Mystem()
sw = stopwords.words('russian')

with open('liza.txt', 'r', encoding='utf8') as f:
    text = f.readlines()

new_lines = []

for line in tqdm(text):
    line = ' '.join([w for w in line.split() if w not in sw])
    newline = ''.join(m.lemmatize(line))
    new_lines.append(newline)
        
with open('liza_lem.txt', 'w', encoding='utf8') as f1:
    for line in new_lines:
        f1.write(line)
```

    100%|████████████████████████████████████████████| 3/3 [00:08<00:00,  2.70s/it]
    

На вход модели даем текстовый файл, каждое предложение на отдельной строчке. Вот игрушечный пример с текстом «Бедной Лизы». Он заранее очищен от пунктуации, приведен к нижнему регистру и лемматизирован.


```python
f = 'liza_lem.txt'
data = gensim.models.word2vec.LineSentence(f)
```

Инициализируем модель. Основные параметры:

+ данные должны быть итерируемым объектом
+ size — размер вектора,
+ window — размер окна наблюдения,
+ min_count — мин. частотность слова в корпусе,
+ sg — используемый алгоритм обучения (0 — CBOW, 1 — Skip-gram),
+ sample — порог для downsampling'a высокочастотных слов,
+ workers — количество потоков,
+ alpha — learning rate,
+ iter — количество итераций,
+ max_vocab_size — позволяет выставить ограничение по памяти при создании словаря (т.е. если ограничение привышается, то низкочастотные слова будут выбрасываться). Для сравнения: 10 млн слов = 1Гб RAM.


```python
%time model_liza = gensim.models.Word2Vec(data, size=300, window=5, min_count=2)
```

    C:\ProgramData\Anaconda3\lib\site-packages\gensim\models\base_any2vec.py:743: UserWarning: C extension not loaded, training will be slow. Install a C compiler and reinstall gensim for fast training.
      "C extension not loaded, training will be slow. "
    

    Wall time: 6.1 s
    

Можно нормализовать вектора, тогда модель будет занимать меньше RAM. Однако после этого её нельзя дотренировывать. Здесь используется L2-нормализация: вектора нормализуются так, что если сложить квадраты всех элементов вектора, в сумме получится 1.


```python
model_liza.init_sims(replace=True)
model_path = "liza.bin"

print("Saving model...")
model_liza.wv.save_word2vec_format(model_path, binary=True)
```

    Saving model...
    

Смотрим, сколько в модели слов:


```python
print(len(model_liza.wv.vocab))
```

    478
    


```python
print(sorted([w for w in model_liza.wv.vocab]))
```

    ['анюта', 'армия', 'ах', 'барин', 'бедный', 'белый', 'берег', 'березовый', 'беречь', 'бесчисленный', 'благодарить', 'бледный', 'блеснуть', 'блестящий', 'близ', 'бог', 'богатый', 'большой', 'бояться', 'брать', 'бросать', 'бросаться', 'бывать', 'быть', 'важный', 'ввечеру', 'вдова', 'велеть', 'великий', 'великолепный', 'верить', 'верно', 'весело', 'веселый', 'весна', 'вести', 'весь', 'весьма', 'ветвь', 'ветер', 'вечер', 'взглядывать', 'вздох', 'вздыхать', 'взор', 'взять', 'вид', 'видеть', 'видеться', 'видный', 'вместе', 'вода', 'возвращаться', 'воздух', 'война', 'воображать', 'воображение', 'воспоминание', 'восторг', 'восхищаться', 'время', 'все', 'вслед', 'вставать', 'встречаться', 'всякий', 'высокий', 'выть', 'выходить', 'глаз', 'глубокий', 'гнать', 'говорить', 'год', 'голос', 'гора', 'горе', 'горестный', 'горлица', 'город', 'горький', 'господь', 'гром', 'грусть', 'давать', 'давно', 'далее', 'дверь', 'движение', 'двор', 'девушка', 'дело', 'день', 'деньги', 'деревня', 'деревянный', 'десять', 'добро', 'добрый', 'довольно', 'доживать', 'долго', 'должный', 'дом', 'домой', 'дочь', 'древний', 'друг', 'другой', 'дуб', 'думать', 'душа', 'едва', 'ехать', 'жалобный', 'желание', 'желать', 'жениться', 'жених', 'женщина', 'жестокий', 'живой', 'жизнь', 'жить', 'забава', 'заблуждение', 'забывать', 'завтра', 'задумчивость', 'закраснеться', 'закричать', 'заря', 'здешний', 'здравствовать', 'зеленый', 'земля', 'златой', 'знать', 'ибо', 'играть', 'идти', 'имя', 'искать', 'исполняться', 'испугаться', 'история', 'исчезать', 'кабинет', 'казаться', 'какой', 'капля', 'карета', 'карман', 'картина', 'катиться', 'келья', 'клятва', 'колено', 'копейка', 'который', 'красота', 'крест', 'крестьянин', 'крестьянка', 'кровь', 'кроме', 'кто', 'купить', 'ландыш', 'ласка', 'ласковый', 'левый', 'лес', 'лететь', 'летний', 'лето', 'лиза', 'лизин', 'лизина', 'лицо', 'лишний', 'лодка', 'ложиться', 'луг', 'луч', 'любезный', 'любить', 'любовь', 'лютый', 'матушка', 'мать', 'место', 'месяц', 'мечта', 'милый', 'мимо', 'минута', 'многочисленный', 'могила', 'мой', 'молить', 'молиться', 'молния', 'молодой', 'молодость', 'молчать', 'монастырь', 'море', 'москва', 'москва-река', 'мочь', 'мрак', 'мрачный', 'муж', 'мы', 'мысль', 'наглядеться', 'надеяться', 'надлежать', 'надобно', 'называть', 'наступать', 'натура', 'находить', 'наш', 'небесный', 'небо', 'невинность', 'невинный', 'неделя', 'нежели', 'нежный', 'незнакомец', 'некоторый', 'непорочность', 'неприятель', 'несколько', 'никакой', 'никто', 'новый', 'ночь', 'обижать', 'облако', 'обманывать', 'обморок', 'образ', 'обращаться', 'обстоятельство', 'объятие', 'огонь', 'один', 'однако', 'окно', 'окрестности', 'он', 'она', 'они', 'оно', 'опираться', 'описывать', 'опустеть', 'освещать', 'оставаться', 'оставлять', 'останавливать', 'останавливаться', 'отвечать', 'отдавать', 'отец', 'отечество', 'отменно', 'отрада', 'очень', 'падать', 'память', 'пастух', 'первый', 'перемениться', 'переставать', 'песня', 'петь', 'печальный', 'писать', 'питать', 'плакать', 'побежать', 'побледнеть', 'погибать', 'подавать', 'подгорюниваться', 'подле', 'подозревать', 'подымать', 'поехать', 'пойти', 'показываться', 'поклониться', 'покойный', 'покрывать', 'покрываться', 'покупать', 'полагать', 'поле', 'помнить', 'поселянин', 'последний', 'постой', 'потуплять', 'поцеловать', 'поцелуй', 'правый', 'представляться', 'прежде', 'преклонять', 'прекрасный', 'прелестный', 'приводить', 'прижимать', 'принадлежать', 'принуждать', 'природа', 'приходить', 'приятно', 'приятный', 'провожать', 'продавать', 'проливать', 'простой', 'просыпаться', 'проходить', 'проч', 'прощать', 'прощаться', 'пруд', 'птичка', 'пылать', 'пять', 'работа', 'работать', 'радость', 'рассказывать', 'расставаться', 'рвать', 'ребенок', 'река', 'решаться', 'робкий', 'роза', 'розовый', 'роман', 'российский', 'роща', 'рубль', 'рука', 'сам', 'самый', 'свет', 'светиться', 'светлый', 'свидание', 'свирель', 'свободно', 'свое', 'свой', 'свойство', 'сделать', 'сделаться', 'сей', 'сердечный', 'сердце', 'сидеть', 'сие', 'сиять', 'сказать', 'сказывать', 'сквозь', 'скорбь', 'скоро', 'скрываться', 'слабый', 'слеза', 'слезать', 'слово', 'случаться', 'слушать', 'слышать', 'смерть', 'сметь', 'смотреть', 'собственный', 'соглашаться', 'солнце', 'спасать', 'спокойно', 'спокойствие', 'спрашивать', 'стадо', 'становиться', 'стараться', 'старуха', 'старушка', 'старый', 'статься', 'стена', 'сто', 'столь', 'стон', 'стонать', 'сторона', 'стоять', 'страшно', 'страшный', 'судьба', 'схватывать', 'счастие', 'счастливый', 'сын', 'таить', 'такой', 'твой', 'темный', 'тения', 'тихий', 'тихонько', 'томный', 'тот', 'трава', 'трепетать', 'трогать', 'ты', 'убивать', 'уверять', 'увидеть', 'увидеться', 'удерживать', 'удивляться', 'удовольствие', 'узнавать', 'улица', 'улыбка', 'уметь', 'умирать', 'унылый', 'упасть', 'услышать', 'утешение', 'утро', 'хижина', 'хлеб', 'ходить', 'холм', 'хороший', 'хотеть', 'хотеться', 'хотя', 'худо', 'худой', 'царь', 'цветок', 'целовать', 'час', 'часто', 'человек', 'чистый', 'читатель', 'чувствительный', 'чувство', 'чувствовать', 'чулок', 'шестой', 'шум', 'шуметь', 'щадить', 'щека', 'эраст', 'эрастов', 'это', 'я']
    

И чему же мы ее научили? Попробуем оценить модель вручную, порешав примеры. Несколько дано ниже, попробуйте придумать свои.


```python
print(model_liza.wv.most_similar(positive=["смерть", "любовь"], negative=["печальный"], topn=1))

print(model_liza.wv.most_similar("любовь", topn=3))

print(model_liza.wv.similarity("лиза", "эраст"))

print(model_liza.wv.doesnt_match("скорбь грусть слеза улыбка".split()))

print(model_liza.wv.words_closer_than("лиза", "эраст"))
```

    [('забава', 0.20675304532051086)]
    [('лиза', 0.27406176924705505), ('спокойствие', 0.23087206482887268), ('рука', 0.17897672951221466)]
    0.056837812
    грусть
    ['свой', 'который', 'мочь', 'сказать', 'ах', 'сей', 'сердце', 'глаз', 'мой', 'любить', 'мать', 'твой', 'хотеть', 'день', 'рука', 'быть', 'друг', 'знать', 'думать', 'всякий', 'один', 'душа', 'жить', 'любезный', 'молодой', 'цветок', 'матушка', 'смотреть', 'старушка', 'лизин', 'земля', 'добрый', 'взять', 'видеть', 'приходить', 'бедный', 'любовь', 'москва', 'новый', 'образ', 'тот', 'чувство', 'скоро', 'прощаться', 'бывать', 'место', 'стоять', 'луч', 'умирать', 'нежный', 'хижина', 'давать', 'плакать', 'увидеть', 'прежде', 'надобно', 'пойти', 'минута', 'берег', 'луг', 'монастырь', 'река', 'удовольствие', 'лицо', 'казаться', 'это', 'возвращаться', 'должный', 'объятие', 'забывать', 'бояться', 'никто', 'город', 'роща', 'находить', 'приятный', 'они', 'стадо', 'веселый', 'сие', 'прекрасный', 'отец', 'переставать', 'пять', 'рубль', 'голос', 'чувствовать', 'мысль', 'последний', 'дело', 'дуб', 'несколько', 'дом', 'солнце', 'кроме', 'деньги', 'муж', 'становиться', 'ночь', 'ходить', 'стараться', 'вид', 'домой', 'рассказывать', 'вечер', 'тихий', 'очень', 'радость', 'мы', 'ласковый', 'барин', 'просыпаться', 'вздыхать', 'утро', 'играть', 'поцелуй', 'жестокий', 'страшный', 'представляться', 'светлый', 'хлеб', 'петь', 'природа', 'отечество', 'судьба', 'березовый', 'принуждать', 'прижимать', 'я', 'ребенок', 'таить', 'покойный', 'копейка', 'другой', 'сам', 'горе', 'обращаться', 'никакой', 'задумчивость', 'крестьянин', 'богатый', 'большой', 'вставать', 'гнать', 'скрываться', 'описывать', 'наглядеться', 'нежели', 'невинный', 'бросаться', 'завтра', 'упасть', 'приводить', 'лето', 'некоторый', 'гора', 'пылать', 'российский', 'летний', 'покрывать', 'выть', 'ветер', 'темный', 'стон', 'воздух', 'горький', 'падать', 'неприятель', 'лютый', 'трогать', 'поселянин', 'щадить', 'молодость', 'молить', 'верно', 'ложиться', 'покрываться', 'закраснеться', 'потуплять', 'наступать', 'бросать', 'испугаться', 'здравствовать', 'деревянный', 'свойство', 'проч', 'молния', 'блеснуть', 'статься', 'роман', 'горлица', 'роза', 'решаться', 'эрастов', 'подгорюниваться', 'светиться', 'свирель', 'шум', 'сметь', 'хотя', 'близ', 'сердечный', 'сиять', 'глубокий', 'сын', 'погибать', 'гром', 'убивать', 'обморок', 'велеть', 'беречь', 'слышать', 'ехать', 'вода', 'двор', 'женщина']
    

#### Как использовать готовую модель

#### RusVectōrēs

На сайте RusVectōrēs (https://rusvectores.org/ru/) собраны предобученные на различных данных модели для русского языка, а также можно поискать наиболее близкие слова к заданному, посчитать семантическую близость нескольких слов и порешать примеры с помощью «калькулятором семантической близости».

Для других языков также можно найти предобученные модели — например, модели [fastText](https://fasttext.cc/docs/en/english-vectors.html) и [GloVe](https://nlp.stanford.edu/projects/glove/)

Ещё давайте посмотрим на векторные романы https://nevmenandr.github.io/novel2vec/

#### Работа с моделью

Модели word2vec бывают разных форматов:

+ .vec.gz — обычный файл
+ .bin.gz — бинарник

Загружаются они с помощью одного и того же гласса `KeyedVectors`, меняется только параметр `binary` у функции `load_word2vec_format`.

Если же эмбеддинги обучены не с помощью word2vec, то для загрузки нужно использовать функцию `load`. Т.е. для загрузки предобученных эмбеддингов `glove`, `fasttext`, `bpe` и любых других нужна именно она.

Скачаем с RusVectōrēs модель для русского языка, обученную на НКРЯ образца 2015 г.


```python
urllib.request.urlretrieve("http://rusvectores.org/static/models/rusvectores2/ruscorpora_mystem_cbow_300_2_2015.bin.gz", "ruscorpora_mystem_cbow_300_2_2015.bin.gz")
```


```python
m = 'ruscorpora_mystem_cbow_300_2_2015.bin.gz'

if m.endswith('.vec.gz'):
    model = gensim.models.KeyedVectors.load_word2vec_format(m, binary=False)
elif m.endswith('.bin.gz'):
    model = gensim.models.KeyedVectors.load_word2vec_format(m, binary=True)
else:
    model = gensim.models.KeyedVectors.load(m)
```

    2019-09-23 12:06:14,681 : INFO : loading projection weights from ruscorpora_mystem_cbow_300_2_2015.bin.gz
    2019-09-23 12:06:14,690 : WARNING : this function is deprecated, use smart_open.open instead
    2019-09-23 12:07:01,110 : INFO : loaded (281776, 300) matrix from ruscorpora_mystem_cbow_300_2_2015.bin.gz
    


```python
words = ['день_S', 'ночь_S', 'человек_S', 'семантика_S', 'студент_S', 'биткоин_S']
```

Частеречные тэги нужны, поскольку это специфика скачанной модели - она была натренирована на словах, аннотированных их частями речи (и лемматизированных). NB! В названиях моделей на `rusvectores` указано, какой тегсет они используют (mystem, upos и т.д.)

Попросим у модели 10 ближайших соседей для каждого слова и коэффициент косинусной близости для каждого:



```python
for word in words:
    # есть ли слово в модели? 
    if word in model:
        print(word)
        # смотрим на вектор слова (его размерность 300, смотрим на первые 10 чисел)
        print(model[word][:10])
        # выдаем 10 ближайших соседей слова:
        for i in model.most_similar(positive=[word], topn=10):
            # слово + коэффициент косинусной близости
            print(i[0], i[1])
        print('\n')
    else:
        # Увы!
        print('Увы, слова "%s" нет в модели!' % word)
```

    день_S
    [-0.02580778  0.00970898  0.01941961 -0.02332282  0.02017624  0.07275085
     -0.01444375  0.03316632  0.01242602  0.02833412]
    

    2019-09-23 12:09:02,405 : INFO : precomputing L2-norms of word weight vectors
    

    неделя_S 0.7165195941925049
    месяц_S 0.6310489177703857
    вечер_S 0.5828739404678345
    утро_S 0.5676207542419434
    час_S 0.5605547428131104
    минута_S 0.5297020077705383
    гекатомбеон_S 0.4897990822792053
    денек_S 0.48224717378616333
    полчаса_S 0.48217129707336426
    ночь_S 0.478074848651886
    
    
    ночь_S
    [-0.00688948  0.00408364  0.06975466 -0.00959525  0.0194835   0.04057068
     -0.00994112  0.06064967 -0.00522624  0.00520327]
    вечер_S 0.6946249008178711
    утро_S 0.5730193257331848
    ноченька_S 0.5582467317581177
    рассвет_S 0.5553584098815918
    ночка_S 0.5351512432098389
    полдень_S 0.5334426164627075
    полночь_S 0.478694349527359
    день_S 0.478074848651886
    сумерки_S 0.4390218257904053
    фундерфун_S 0.4340824782848358
    
    
    человек_S
    [ 0.02013756 -0.02670703 -0.02039861 -0.05477146  0.00086402 -0.01636335
      0.04240306 -0.00025525 -0.14045681  0.04785006]
    женщина_S 0.5979775190353394
    парень_S 0.4991787374019623
    мужчина_S 0.4767409861087799
    мужик_S 0.47383999824523926
    россиянин_S 0.47190430760383606
    народ_S 0.46547412872314453
    согражданин_S 0.45378512144088745
    горожанин_S 0.44368088245391846
    девушка_S 0.4431447982788086
    иностранец_S 0.43849870562553406
    
    
    семантика_S
    [-0.03066749  0.0053851   0.1110732   0.0152335   0.00440643  0.00384104
      0.00096944 -0.03538784 -0.00079585  0.03220548]
    семантический_A 0.5334584712982178
    понятие_S 0.5030269026756287
    сочетаемость_S 0.4817051589488983
    актант_S 0.47596412897109985
    хронотоп_S 0.46330294013023376
    метафора_S 0.46158888936042786
    мышление_S 0.46101194620132446
    парадигма_S 0.45796656608581543
    лексема_S 0.45688071846961975
    смысловой_A 0.4543077349662781
    
    
    студент_S
    [ 0.02558023  0.0529849  -0.07036145  0.00279281 -0.09874777 -0.01620521
     -0.03918766  0.0326411   0.09191283  0.03495219]
    преподаватель_S 0.6958175301551819
    аспирант_S 0.6589953899383545
    выпускник_S 0.6523088216781616
    студентка_S 0.6321653723716736
    профессор_S 0.6080018281936646
    курсистка_S 0.5818493962287903
    юрфак_S 0.580669105052948
    первокурсник_S 0.5805511474609375
    семинарист_S 0.5773230791091919
    гимназист_S 0.5747809410095215
    
    
    Увы, слова "биткоин_S" нет в модели!
    

Находим косинусную близость пары слов:


```python
print(model.similarity('человек_S', 'обезьяна_S'))
```

    0.23895612
    

Что получится, если вычесть из пиццы Италию и прибавить Сибирь?

+ positive — вектора, которые мы складываем
+ negative — вектора, которые вычитаем


```python
print(model.most_similar(positive=['пицца_S', 'сибирь_S'], negative=['италия_S'])[0][0])
```

    пельмень_S
    

Найди лишнее!


```python
print(model.doesnt_match('мир_S жизнь_S бытие_S ничто_S'.split()))
```

    2019-09-23 12:12:49,578 : WARNING : vectors for words {'ничто_S'} are not present in the model, ignoring these words
    

    мир_S
    


```python
print(model.most_similar(positive=['жизнь_S'], negative=['любовь_S']))
      
```

    [('быт_S', 0.3733804225921631), ('жисть_S', 0.3260636329650879), ('существование_S', 0.29437491297721863), ('обстановка_S', 0.2809719443321228), ('махаль_S', 0.26148635149002075), ('жизень_S', 0.2572200894355774), ('уклад_S', 0.25639140605926514), ('скарб_S', 0.2556115388870239), ('житие_S', 0.251535028219223), ('передряга_S', 0.2505052387714386)]
    

#### Оценка

Это, конечно, хорошо, но как понять, какая модель лучше? Или вот, например, я сделал свою модель, а как понять, насколько она хорошая?

Для этого существуют специальные датасеты для оценки качества дистрибутивных моделей. Основных два: один измеряет точность решения задач на аналогии (про Россию и пельмени), а второй используется для оценки коэффициента семантической близости.

#### Word Similarity

Этот метод заключается в том, чтобы оценить, насколько представления о семантической близости слов в модели соотносятся с \"представлениями\" людей.

| слово 1    | слово 2    | близость |
|------------|------------|----------|
| кошка      | собака     | 0.7      | 
| чашка      | кружка     | 0.9      | 

Для каждой пары слов из заранее заданного датасета мы можем посчитать косинусное расстояние, и получить список таких значений близости. При этом у нас уже есть список значений близостей, сделанный людьми. Мы можем сравнить эти два списка и понять, насколько они похожи (например, посчитав корреляцию). Эта мера схожести должна говорить о том, насколько модель хорошо моделирует расстояния о слова.

#### Аналогии

Другая популярная задача для "внутренней" оценки называется задачей поиска аналогий. Как мы уже разбирали выше, с помощью простых арифметических операций мы можем модифицировать значение слова. Если заранее собрать набор слов-модификаторов, а также слов, которые мы хотим получить в результаты модификации, то на основе подсчёта количества "попаданий" в желаемое слово мы можем оценить, насколько хорошо работает модель.

В качестве слов-модификатор мы можем использовать семантические аналогии. Скажем, если у нас есть некоторое отношение "страна-столица", то для оценки модели мы можем использовать пары наподобие "Россия-Москва", "Норвегия-Осло", и т.д. Датасет будет выглядеть следующм образом:

| слово 1    | слово 2    | отношение     | 
|------------|------------|---------------|
| Россия     | Москва     | страна-столица| 
| Норвегия   | Осло       | страна-столица|

Рассматривая случайные две пары из этого набора, мы хотим, имея триплет (Россия, Москва, Норвегия) хотим получить слово "Осло", т.е. найти такое слово, которое будет находиться в том же отношении со словом "Норвегия", как "Россия" находится с Москвой.

Датасеты для русского языка можно скачать на странице с моделями на RusVectores. Посчитаем качество нашей модели НКРЯ на датасете про аналогии:


```python
res = model.accuracy('ru_analogy_tagged.txt')
```


```python
print(res[4]['incorrect'][:10])
```

#### Задание 1

+ Возьмите небольшой кусочек текста или стихотворение.
+ Замените все неслужебные слова в нём на их ближайших соседей из нашей модели.
+ Прокомментируйте результат.

#### Задание 2

+ Возьмите интересный Вам текст.
+ Лемматизируйте текст, отчистите от пунктуации и служебной информации и обучите на нем модель word2vec (поэкспериментируйте с размером окна, с длиной вектора). 
+ Найдите по 5 ближайших слов к нескольким интересующим Вас словам. Обязательно попробуйте взять слова различной частеречной принадлежности, различных семантических классов (абстрактные слова, экспрессивы). Учтите, что слова может не быть в модели!
+ Найдите по 5 "далёких" слов к нескольким интересующим Вас словам. Обязательно попробуйте взять слова различной частеречной принадлежности, различных семантических классов (абстрактные слова, экспрессивы).
+ Прокомментируйте результат.