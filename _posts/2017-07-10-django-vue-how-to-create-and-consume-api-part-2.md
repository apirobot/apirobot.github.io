---
title: Django + Vue. Как создать и обработать API. Часть 2
date: 2017-07-10 8:00:00
---

<a target="blank" href="http://apirobot.me/posts/django-vue-how-to-create-and-consume-api-part-1">В предыдущей части</a> урока мы написали бэкэнд для нашего приложения с заметками. В этом уроке мы продолжим, и напишем фронтэнд часть, используя фреймворк <a target="blank" href="https://vuejs.org/">vue.js</a> для Javascript.

Исходный код урока: <a target="blank" href="https://github.com/apirobot/django-vue-simplenote">https://github.com/apirobot/django-vue-simplenote</a>

---

После предыдущего урока, структура вашего приложения должна выглядить примерно так:

```bash
django-vue-simplenote
└── backend
    ├── db.sqlite3
    ├── manage.py
    ├── notes
    │   ├── __init__.py
    │   ├── migrations
    │   │   ├── 0001_initial.py
    │   │   └── __init__.py
    │   ├── models.py
    │   ├── serializers.py
    │   ├── urls.py
    │   └── views.py
    └── simplenote
        ├── __init__.py
        ├── settings.py
        ├── urls.py
        └── wsgi.py
```

## Настройка фронтэнда и установка зависимостей

Давайте начнем с создания шаблона с помощью коммандной утилиты <a target="blank" href="https://github.com/vuejs/vue-cli">vue-cli</a>:

```bash
$ cd django-vue-simplenote
$ vue init webpack simplenote
```

![Vue-cli](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-how-to-create-and-consume-api-part-2/vue-cli.png)

Коммандная утилита создала папку simplenote. Переименуем эту папку:

```bash
$ mv simplenote frontend
```

Устанавливаем зависимости и запускаем сборку:

```bash
$ cd frontend
$ npm install
$ npm run dev
```

Если все прошло успешно, то теперь ваш Vue проект доступен по адресу localhost:8080

Кроме стандартных зависимостей, нам еще в дальнейшем понадобится:

1. HTML препроцессор <a target="blank" href="https://pugjs.org/">pug</a>
2. <a target="blank" href="https://github.com/mzabriskie/axios">Axios</a> для отправки HTTP запросов к нашему API
3. <a target="blank" href="https://vuex.vuejs.org/">Vuex</a> для хранения наших заметок в хранилище

Установим все это:

```bash
$ npm install pug --save-dev
$ npm install axios vue-axios --save
$ npm install vuex --save
```

Чтобы не писать самим свои стили, мы будем использовать CSS фреймворк <a target="blank" href="https://picturepan2.github.io/spectre/">spectre.css</a>. Включим его в наш проект. Для этого достаточно добавить ссылку на cdn в файле frontend/index.html:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>simplenote</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/spectre.css/0.2.14/spectre.min.css">
  </head>
  <body>
    <div id="app"></div>
    <!-- built files will be auto injected -->
  </body>
</html>
```

## API

Прежде чем отобразить заметки на нашей странице, их нужно сначала получить. Для этого создадим несколько функций с запросами к нашему API.

Так как все наши запросы будут обращаться к одному и тому же URL (http://localhost:8000/api/v1/), то мы создадим отдельный экземпляр axios, который будем позже использовать:

```javascript
// frontend/src/api/common.js
import axios from 'axios'

export const HTTP = axios.create({
  baseURL: 'http://localhost:8000/api/v1/'
})
```

В нашем приложении мы должны иметь возможность создать заметку, получить список всех заметок, и удалить заметку. Создадим функции для каждого запроса:

```javascript
// frontend/src/api/notes
import { HTTP } from './common'

export const Note = {
  create (config) {
    return HTTP.post('/notes/', config).then(response => {
      return response.data
    })
  },
  delete (note) {
    return HTTP.delete(`/notes/${note.id}/`)
  },
  list () {
    return HTTP.get('/notes/').then(response => {
      return response.data
    })
  }
}
```

Если вы попытаетесь отправить запрос к API, вызвав одну из функций выше, то вы должны получить ошибку:

![Error](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-how-to-create-and-consume-api-part-2/xmlhttprequest.png)

Проблема в том, что по умолчанию, мы можем отправлять запросы только в пределах своего хоста (localhost:8080). Для того, чтобы решить эту проблему, нужно добавить специальные response headers на нашем сервере. Делается это просто. Устанавливаем <a target="blank" href="https://github.com/ottoyiu/django-cors-headers">django-cors-headers</a>:

```bash
$ pip install django-cors-heders
```

Обновляем настройки проекта django:

```python
# backend/simplenote/settings.py

INSTALLED_APPS = [
    ...
    'corsheaders',
]

MIDDLEWARE = [
    ...
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    ...
]

CORS_ORIGIN_ALLOW_ALL = True
```

## Хранилище

Давайте создадим хранилище для заметок с помощью Vuex. Начнем с простого:

```bash
$ mkdir store
$ touch store/index.js
$ touch store/mutation-types.js
```

Добавим хранилище в файл main.js, чтобы в дальнейшем мы могли его использовать в наших компонентах:

```javascript
// frontend/src/main.js
import Vue from 'vue'
import App from './App'
import store from './store'

Vue.config.productionTip = false

/* eslint-disable no-new */
new Vue({
  el: '#app',
  store,
  template: '<App/>',
  components: { App }
})
```

Добавим типы мутаций:

```javascript
// frontend/src/store/mutation-types.js
export const ADD_NOTE = 'ADD_NOTE'
export const REMOVE_NOTE = 'REMOVE_NOTE'
export const SET_NOTES = 'SET_NOTES'
```

Реализуем хранилище:

```javascript
// frontend/src/store/index.js
import Vue from 'vue'
import Vuex from 'vuex'
import { Note } from '../api/notes'
import {
  ADD_NOTE,
  REMOVE_NOTE,
  SET_NOTES
} from './mutation-types.js'

Vue.use(Vuex)

// Состояние
const state = {
  notes: []  // список заметок
}

// Геттеры
const getters = {
  notes: state => state.notes  // получаем список заметок из состояния
}

// Мутации
const mutations = {
  // Добавляем заметку в список
  [ADD_NOTE] (state, note) {
    state.notes = [note, ...state.notes]
  },
  // Убираем заметку из списка
  [REMOVE_NOTE] (state, { id }) {
    state.notes = state.notes.filter(note => {
      return note.id !== id
    })
  },
  // Задаем список заметок
  [SET_NOTES] (state, { notes }) {
    state.notes = notes
  }
}

// Действия
const actions = {
  createNote ({ commit }, noteData) {
    Note.create(noteData).then(note => {
      commit(ADD_NOTE, note)
    })
  },
  deleteNote ({ commit }, note) {
    Note.delete(note).then(response => {
      commit(REMOVE_NOTE, note)
    })
  },
  getNotes ({ commit }) {
    Note.list().then(notes => {
      commit(SET_NOTES, { notes })
    })
  }
}

export default new Vuex.Store({
  state,
  getters,
  actions,
  mutations
})
```

## Компоненты

Наше приложение будет состоять из одной страницы, которая включает в себя форму с добавлением новой заметки, и список заметок, отсортированных по дате добавления. Поэтому мы создаем два Vue компонента: CreateNote (форма с добавлением заметки) и NoteList (список всех заметок):

```bash
$ touch src/components/CreateNote.vue
$ touch src/components/NoteList.vue
```

Добавим эти компоненты в наш основной компонент src/App.vue вместе с небольшим html и css кодом:

```html
<template lang="pug">
  #app
    section.container.grid-960
      .columns
        .column.col-2
        .column.col-8.col-md-12
          header.text-center
            h2 Create note
          create-note
          header.text-center
            h2 List of notes
          note-list
        .column.col-2
</template>

<script>
import CreateNote from './components/CreateNote'
import NoteList from './components/NoteList'

export default {
  name: 'app',
  components: {
    'create-note': CreateNote,
    'note-list': NoteList
  }
}
</script>

<style>
  @import url(https://fonts.googleapis.com/css?family=Eczar);
  @import url(https://fonts.googleapis.com/css?family=Work+Sans);

  body {
    font-family: "Work Sans", "Segoe UI", "Helvetica Neue", sans-serif;
  }

  h1, h2, h3, h4, h5, h6 {
    font-family: "Eczar", sans-serif;
  }
</style>
```

Реализуем компонент frontend/src/components/CreateNote.vue:

```html
<template lang="pug">
  form.form-horizontal(@submit="submitForm")
    .form-group
      .col-3
        label.form-label Title
      .col-9
        input.form-input(type="text" v-model="title" placeholder="Type note title...")
    .form-group
      .col-3
        label.form-label Body
      .col-9
        textarea.form-input(v-model="body" rows=8 placeholder="Type your note...")
    .form-group
      .col-3
      .col-9
        button.btn.btn-primary(type="submit") Create
</template>

<script>
export default {
  name: 'create-note',
  data () {
    return {
      'title': '',
      'body': ''
    }
  },
  methods: {
    submitForm (event) {
      this.createNote()

      // Т.к. мы уже отправили запрос на создание заметки строчкой выше,
      // нам нужно теперь очистить поля title и body
      this.title = ''
      this.body = ''

      // preventDefault нужно для того, чтобы страница
      // не перезагружалась после нажатия кнопки submit
      event.preventDefault()
    },
    createNote () {
      // Вызываем действие `createNote` из хранилища, которое
      // отправит запрос на создание новой заметки к нашему API.
      this.$store.dispatch('createNote', { title: this.title, body: this.body })
    }
  }
}
</script>
```

Реализуем компонент frontend/src/components/NoteList.vue:

```html
<template lang="pug">
  #app
    .card(v-for="note in notes")
      .card-header
        button.btn.btn-clear.float-right(@click="deleteNote(note)")
        .card-title {{ note.title }}
        .card-subtitle {{ note.created_at }}
      .card-body {{ note.body }}
</template>


<script>
import { mapGetters } from 'vuex'

export default {
  name: 'note-list',
  computed: mapGetters(['notes']),
  methods: {
    deleteNote (note) {
      // Вызываем действие `deleteNote` из нашего хранилища, которое
      // попытается удалить заметку из нашех базы данных, отправив запрос к API
      this.$store.dispatch('deleteNote', note)
    }
  },
  beforeMount () {
    // Перед тем как загрузить страницу, нам нужно получить список всех
    // имеющихся заметок. Для этого мы вызываем действие `getNotes` из
    // нашего хранилища
    this.$store.dispatch('getNotes')
  }
}
</script>

<style>
  header {
    margin-top: 50px;
  }
</style>
```

## Запуск

На этом все. Фронтэнд и бэкэнд готов. Осталось только запустить это все вместе, введя команду в папке с проектом:

```bash
$ npm run --prefix frontend dev & python backend/manage.py runserver
```

Либо откройте два терминала и в одном из них запустите django, а в другом vue:

```bash
$ python backend/manage.py runserver
$ cd frontend && npm run dev
```

## Конец

Проблема? [Используй GitHub Issues](https://github.com/apirobot/django-vue-simplenote).
