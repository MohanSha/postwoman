<template>
  <div class="page">

    <pw-section class="blue" label="Request" ref="request">
      <ul>
        <li>
          <label for="method">Method</label>
          <select id="method" v-model="method">
            <option>GET</option>
            <option>POST</option>
            <option>PUT</option>
            <option>DELETE</option>
            <option>OPTIONS</option>
          </select>
        </li>
        <li>
          <label for="url">URL</label>
          <input id="url" type="url" v-bind:class="{ error: !isValidURL }" v-model="url" v-on:keyup.enter="sendRequest">
        </li>
        <li>
          <label for="path">Path</label>
          <input id="path" v-model="path" v-on:keyup.enter="sendRequest">
        </li>
        <li>
          <label for="action">&nbsp;</label>
          <button id="action" name="action" @click="sendRequest" :disabled="!isValidURL">Send</button>
        </li>
      </ul>
    </pw-section>

    <pw-section class="blue-dark" label="Request Body" v-if="method === 'POST' || method === 'PUT'">
      <ul>
        <li>
          <label>Content Type</label>
          <select v-model="contentType">
            <option>application/json</option>
            <option>www-form/urlencoded</option>
          </select>
          <span>
            <input v-model="rawInput" style="cursor: pointer;" type="checkbox" id="rawInput">
            <label for="rawInput" style="cursor: pointer;">Raw Input</label>
          </span>
        </li>
      </ul>
      <div v-if="!rawInput">
        <ol v-for="(param, index) in bodyParams">
          <li>
            <label :for="'bparam'+index">Key {{index + 1}}</label>
            <input :name="'bparam'+index" v-model="param.key">
          </li>
          <li>
            <label :for="'bvalue'+index">Value {{index + 1}}</label>
            <input :name="'bvalue'+index" v-model="param.value">
          </li>
          <li>
            <label for="request">&nbsp;</label>
            <button name="request" @click="removeRequestBodyParam(index)">Remove</button>
          </li>
        </ol>
        <ul>
          <li>
            <label for="addrequest">Action</label>
            <button name="addrequest" @click="addRequestBodyParam">Add</button>
          </li>
        </ul>
        <ul>
          <li>
            <label for="request">Parameter List</label>
            <textarea name="request" rows="1" readonly>{{rawRequestBody || '(add at least one parameter)'}}</textarea>
          </li>
        </ul>
      </div><div v-else>
        <textarea v-model="rawParams" style="font-family: monospace;" rows="16" @keydown="formatRawParams"></textarea>
      </div>
    </pw-section>

    <pw-section class="green" label="Authentication" collapsed>
      <ul>
        <li>
          <label for="auth">Authentication Type</label>
          <select v-model="auth">
            <option>None</option>
            <option>Basic</option>
            <option>Bearer Token</option>
          </select>
        </li>
      </ul>
      <ul v-if="auth === 'Basic'">
        <li>
          <label for="http_basic_user">User</label>
          <input v-model="httpUser">
        </li>
        <li>
          <label for="http_basic_passwd">Password</label>
          <input v-model="httpPassword" type="password">
        </li>
      </ul>
      <ul v-if="auth === 'Bearer Token'">
        <li>
          <label for="bearer_token">Token</label>
          <input v-model="bearerToken">
        </li>
      </ul>
    </pw-section>

    <pw-section class="cyan" label="Parameters" collapsed>
      <ol v-for="(param, index) in params">
        <li>
          <label :for="'param'+index">Key {{index + 1}}</label>
          <input :name="'param'+index" v-model="param.key">
        </li>
        <li>
          <label :for="'value'+index">Value {{index + 1}}</label>
          <input :name="'value'+index" v-model="param.value">
        </li>
        <li>
          <label for="param">&nbsp;</label>
          <button name="param" @click="removeRequestParam(index)">Remove</button>
        </li>
      </ol>
      <ul>
        <li>
          <label for="add">Action</label>
          <button name="add" @click="addRequestParam">Add</button>
        </li>
      </ul>
      <ul>
        <li>
          <label for="request">Parameter List</label>
          <textarea name="request" rows="1" readonly>{{queryString || '(add at least one parameter)'}}</textarea>
        </li>
      </ul>
    </pw-section>

    <pw-section class="purple" label="Response" id="response" ref="response">
      <ul>
        <li>
          <label for="status">status</label>
          <input name="status" type="text" readonly :value="response.status || '(waiting to send request)'" :class="statusCategory ? statusCategory.className : ''">
        </li>
      </ul>
      <ul v-for="(value, key) in response.headers">
        <li>
          <label for="value">{{key}}</label>
          <input name="value" :value="value" readonly>
        </li>
      </ul>
      <ul>
        <li> 
        <div class="flex-wrap">
          <label for="body">response</label>
          <button v-if="response.body" name="action" class="btn-copy" @click="copyResponse">Copy Response</button>
         </div>
          <textarea name="body" rows="10" id="response-details" readonly>{{response.body || '(waiting to send request)'}}</textarea>
        </li>
      </ul>
    </pw-section>

    <pw-section class="gray" label="History">
      <ul>
        <li>
          <button v-bind:class="{ disabled: noHistoryToClear }" v-on:click="clearHistory">Clear History</button>
        </li>
      </ul>
      <ul v-for="entry in history">
        <li>
          <label for="time">Time</label>
          <input name="time" type="text" readonly :value="entry.time">
        </li>
        <li>
          <label for="name">Method</label>
          <input name="name" type="text" readonly :value="entry.method">
        </li>
        <li>
          <label for="name">URL</label>
          <input name="name" type="text" readonly :value="entry.url">
        </li>
        <li>
          <label for="name">Path</label>
          <input name="name" type="text" readonly :value="entry.path">
        </li>
        <li>
          <label for="delete">&nbsp;</label>
          <button name="delete" @click="deleteHistory(entry)">Delete</button>
        </li>
        <li>
          <label for="use">&nbsp;</label>
          <button name="use" @click="useHistory(entry)">Use</button>
        </li>
      </ul>
    </pw-section>

  </div>
</template>

<script>
  const parseHeaders = xhr => {
      const headers = xhr.getAllResponseHeaders().trim().split(/[\r\n]+/);
      const headerMap = {};
      headers.forEach(line => {
          const parts = line.split(': ');
          const header = parts.shift().toLowerCase();
          const value = parts.join(': ');
          headerMap[header] = value
      });
      return headerMap
  };

  import section from "../components/section";

  export default {
    components: {
      'pw-section': section
    },

      data () {
          return {
              method: 'GET',
              url: 'https://reqres.in',
              auth: 'None',
              path: '/api/users',
              httpUser: '',
              httpPassword: '',
              bearerToken: '',
              params: [],
              bodyParams: [],
              rawParams: '',
              rawInput: false,
              contentType: 'application/json',
              response: {
                  status: '',
                  headers: '',
                  body: ''
              },
              history: window.localStorage.getItem('history') ? JSON.parse(window.localStorage.getItem('history')) : []
          }
      },
      computed: {
          statusCategory(){
            return [
              {name: 'informational', statusCodeRegex: new RegExp(/[1][0-9]+/), className: 'info-response'},
              {name: 'successful', statusCodeRegex: new RegExp(/[2][0-9]+/), className: 'success-response'},
              {name: 'redirection', statusCodeRegex: new RegExp(/[3][0-9]+/), className: 'redir-response'},
              {name: 'client error', statusCodeRegex: new RegExp(/[4][0-9]+/), className: 'cl-error-response'},
              {name: 'server error', statusCodeRegex: new RegExp(/[5][0-9]+/), className: 'sv-error-response'},
            ].find(status => status.statusCodeRegex.test(this.response.status));
          },
          noHistoryToClear() {
              return this.history.length === 0;
          },
          isValidURL() {
            const protocol = '^(https?:\\/\\/)?';

            const validIP = new RegExp(protocol + "(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])$");
            const validHostname = new RegExp(protocol + "(([a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9\-]*[a-zA-Z0-9])\.)*([A-Za-z0-9]|[A-Za-z0-9][A-Za-z0-9\-]*[A-Za-z0-9])$");

            return validIP.test(this.url) || validHostname.test(this.url);
          },
          rawRequestBody() {
              const {
                  bodyParams
              } = this
              if (this.contentType === 'application/json') {
                  try {
                      const obj = JSON.parse(`{${bodyParams.filter(({ key }) => !!key).map(({ key, value }) => `
              "${key}": "${value}"
              `).join()}}`)
                      return JSON.stringify(obj)
                  } catch (ex) {
                      return 'invalid'
                  }
              } else {
                  return bodyParams
                      .filter(({
                                   key
                               }) => !!key)
                      .map(({
                                key,
                                value
                            }) => `${key}=${encodeURIComponent(value)}`).join('&')
              }
          },
          queryString() {
              const result = this.params
                  .filter(({
                               key
                           }) => !!key)
                  .map(({
                            key,
                            value
                        }) => `${key}=${encodeURIComponent(value)}`).join('&')
              return result == '' ? '' : `?${result}`
          }
      },
      methods: {
          deleteHistory(entry) {
              this.history.splice(this.history.indexOf(entry), 1)
              window.localStorage.setItem('history', JSON.stringify(this.history))
          },
          clearHistory() {
              this.history = []
              window.localStorage.setItem('history', JSON.stringify(this.history))
          },
          useHistory({
                         method,
                         url,
                         path
                     }) {
              this.method = method
              this.url = url
              this.path = path
              this.$refs.request.$el.scrollIntoView({
                  behavior: 'smooth'
              })
          },
          sendRequest() {
              if (!this.isValidURL) {
                  alert('Please check the formatting of the URL');
                  return
              }
              const n = new Date().toLocaleTimeString()
              this.history = [{
                  time: n,
                  method: this.method,
                  url: this.url,
                  path: this.path
              }, ...this.history]
              window.localStorage.setItem('history', JSON.stringify(this.history))
              if (this.$refs.response.$el.classList.contains('hidden')) {
                  this.$refs.response.$el.classList.toggle('hidden')
              }
              this.$refs.response.$el.scrollIntoView({
                  behavior: 'smooth'
              })
              this.response.status = 'Fetching...'
              this.response.body = 'Loading...'
              const xhr = new XMLHttpRequest()
              const user = this.auth === 'Basic' ? this.httpUser : null
              const pswd = this.auth === 'Basic' ? this.httpPassword : null
              xhr.open(this.method, this.url + this.path + this.queryString, true, user, pswd)
              if (this.auth === 'Bearer Token') {
                  xhr.setRequestHeader('Authorization', 'Bearer ' + this.bearerToken);
              }
              if (this.method === 'POST' || this.method === 'PUT') {
                  const requestBody = this.rawInput ? this.rawParams : this.rawRequestBody;
                  xhr.setRequestHeader('Content-Length', requestBody.length)
                  xhr.setRequestHeader('Content-Type', `${this.contentType}; charset=utf-8`)
                  xhr.send(requestBody)
              } else {
                  xhr.send()
              }
              xhr.onload = e => {
                  this.response.status = xhr.status
                  const headers = this.response.headers = parseHeaders(xhr)
                  if ((headers['content-type'] || '').startsWith('application/json')) {
                      this.response.body = JSON.stringify(JSON.parse(xhr.responseText), null, 2)
                  } else {
                      this.response.body = xhr.responseText
                  }
              }
              xhr.onerror = e => {
                  this.response.status = xhr.status
                  this.response.body = xhr.statusText
              }
          },
          addRequestParam() {
              this.params.push({
                  key: '',
                  value: ''
              })
              return false
          },
          removeRequestParam(index) {
              this.params.splice(index, 1)
          },
          addRequestBodyParam() {
              this.bodyParams.push({
                  key: '',
                  value: ''
              })
              return false
          },
          removeRequestBodyParam(index) {
              this.bodyParams.splice(index, 1)
          },
          formatRawParams(event) {
            if ((event.which !== 13 && event.which !== 9)) {
              return;
            }
            const textBody = event.target.value;
            const textBeforeCursor = textBody.substring(0, event.target.selectionStart);
            const textAfterCursor = textBody.substring(event.target.selectionEnd);

            if (event.which === 13) {
              event.preventDefault();
              const oldSelectionStart = event.target.selectionStart;
              const lastLine = textBeforeCursor.split('\n').slice(-1)[0];
              const rightPadding = lastLine.match(/([\s\t]*).*/)[1] || "";
              event.target.value = textBeforeCursor + '\n' + rightPadding + textAfterCursor;
              setTimeout(() => event.target.selectionStart = event.target.selectionEnd =  oldSelectionStart + rightPadding.length + 1, 1);
            }
            else if (event.which === 9) {
              event.preventDefault();
              const oldSelectionStart = event.target.selectionStart;
              event.target.value = textBeforeCursor + '\xa0\xa0' + textAfterCursor;
              event.target.selectionStart = event.target.selectionEnd = oldSelectionStart + 2;
              return false;
            }

          },
           copyResponse() {
            var copyText = document.getElementById("response-details");
            copyText.select();
            document.execCommand("copy");
          }
      }
  }
</script>
