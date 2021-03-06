<template>
  <div class="container">
    <div class="row">
      <div class="col-xs-12 form-inline">
        Template:
        <select class="form-control" v-model="templateIndex">
          <option v-for="(template, index) in templates" :value="index">{{ template.name }}</option>
        </select>
      </div>
    </div>

    <hr>

    <div class="row">
      <div class="col-md-4">
        <editor :template="currentTemplate" @send="send" @change="change" @reset="reset"></editor>
      </div>
      <div class="col-md-8">
        <div v-if="currentTemplate">{{ currentTemplate.name }}</div>
        <pre v-html="highlightedTemplate"></pre>
        <pre v-html="highlightedResponse"></pre>
      </div>
    </div>
  </div>
</template>

<script>
import Promise from 'promise-polyfill'
if (!window.Promise) {
  window.Promise = Promise
}
import 'whatwg-fetch'

import Prism from 'prismjs'
import 'prismjs/themes/prism.css'
import 'prismjs/components/prism-markup'

import nunjucks from 'nunjucks'

import beautify from 'js-beautify'

import Editor from './Editor'

export default {
  data () {
    return {
      templates: [],
      templateIndex: null,
      currentTemplate: null,
      xmlTemplate: '',
      response: '',
      data: {
      },
      url: null,
      apiKey: null
    }
  },

  created () {
    fetch('./static/api_templates.json')
      .then((response) => response.json())
      .then((json) => {
        this.templates = json.templates

        // preselect template based on the url hash template, e.g.
        // http://localhost:8080/#template=test.xml
        var splits = window.location.hash.substr(1).split('=')
        if (splits[0] === 'template') {
          json.templates.forEach((template, index) => {
            if (template.template === splits[1]) {
              this.templateIndex = index
            }
          })
        }
        if (this.templateIndex === null) {
          this.templateIndex = 0
        }
      })
      .catch((ex) => (console.log('parsing api_templates failed', ex)))

    fetch('./static/config.json')
      .then((response) => response.json())
      .then((json) => {
        this.url = json.url
        this.apiKey = json.api_key
      })

    nunjucks.configure({trimBlocks: true, lstripBlocks: true})
  },

  watch: {
    templateIndex (newIndex) {
      this.currentTemplate = this.templates[this.templateIndex]
      fetch('./static/' + this.currentTemplate.template)
          .then((response) => response.text())
          .then((xml) => (this.xmlTemplate = xml))
          .catch((ex) => (console.log(`loading of xml template ${this.currentTemplate.template} failed`)))
    }
  },

  computed: {
    renderedTemplate () {
      return nunjucks.renderString(this.xmlTemplate, this.data)
    },
    highlightedTemplate () {
      return Prism.highlight(beautify.html(this.renderedTemplate), Prism.languages.markup)
    },
    highlightedResponse () {
      return Prism.highlight(beautify.html(this.response), Prism.languages.markup)
    }
  },

  methods: {
    send () {
      fetch(this.url, {
        method: 'POST',
        headers: {
          'Content-Type': 'text/xml',
          'Authorization': this.apiKey
        },
        body: this.renderedTemplate
      })
      .then((response) => response.text())
      .then((xml) => (this.response = xml))
      .catch((ex) => (console.log(`api request failed`, ex)))
    },
    change (newData) {
      this.data = newData
    },
    reset () {
      this.response = ''
    }
  },

  components: {
    Editor
  }
}
</script>
