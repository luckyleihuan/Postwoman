<template>
  <div class="page">
    <div class="content" ref="request">
      <request-section
        :historyComponent="this.$refs.historyComponent"
        :store="this.$store"
        :receiveRequest="this.receiveRequest">
      </request-section>
      <aside class="sticky-inner inner-right">
        <section>
          <input id="history-tab" type="radio" name="side" checked="checked" />
          <label for="history-tab">History</label>
          <div class="tab">
            <history
              @useHistory="handleUseHistory"
              ref="historyComponent"
            ></history>
          </div>
          <input id="collection-tab" type="radio" name="side" />
          <label for="collection-tab">Collections</label>
          <div class="tab">
            <pw-section class="yellow" label="Collections" ref="collections">
              <collections />
            </pw-section>
          </div>
        </section>
      </aside>

      <save-request-as
        v-bind:show="showRequestModal"
        v-on:hide-model="hideRequestModal"
        v-bind:editing-request="editRequest"
      ></save-request-as>

      <pw-modal v-if="showModal" @close="showModal = false">
        <div slot="header">
          <ul>
            <li>
              <div class="flex-wrap">
                <h3 class="title">Import cURL</h3>
                <div>
                  <button class="icon" @click="showModal = false">
                    <i class="material-icons">close</i>
                  </button>
                </div>
              </div>
            </li>
          </ul>
        </div>
        <div slot="body">
          <ul>
            <li>
              <textarea
                id="import-text"
                autofocus
                rows="8"
                placeholder="Enter cURL"
              ></textarea>
            </li>
          </ul>
        </div>
        <div slot="footer">
          <ul>
            <li>
              <button class="icon" @click="handleImport">
                <i class="material-icons">get_app</i>
                <span>Import</span>
              </button>
            </li>
          </ul>
        </div>
      </pw-modal>

      <pw-modal v-if="!isHidden" @close="isHidden = true">
        <div slot="header">
          <ul>
            <li>
              <div class="flex-wrap">
                <h3 class="title">Generate code</h3>
                <div>
                  <button class="icon" @click="isHidden = true">
                    <i class="material-icons">close</i>
                  </button>
                </div>
              </div>
            </li>
          </ul>
        </div>
        <div slot="body">
          <ul>
            <li>
              <label for="requestType">Request Type</label>
              <select id="requestType" v-model="requestType">
                <option>JavaScript XHR</option>
                <option>Fetch</option>
                <option>cURL</option>
              </select>
            </li>
          </ul>
          <ul>
            <li>
              <div class="flex-wrap">
                <label for="generatedCode">Generated Code</label>
                <div>
                  <button
                    class="icon"
                    @click="copyRequestCode"
                    id="copyRequestCode"
                    ref="copyRequestCode"
                    v-tooltip="'Copy code'"
                  >
                    <i class="material-icons">file_copy</i>
                  </button>
                </div>
              </div>
              <textarea
                id="generatedCode"
                ref="generatedCode"
                name="generatedCode"
                rows="8"
                v-model="requestCode"
              ></textarea>
            </li>
          </ul>
        </div>
        <div slot="footer"></div>
      </pw-modal>
    </div>
  </div>
</template>
<script>
import section from "../components/section";
import url from "url";
import querystring from "querystring";
import textareaAutoHeight from "../directives/textareaAutoHeight";
import parseCurlCommand from "../assets/js/curlparser.js";
import getEnvironmentVariablesFromScript from "../functions/preRequest";
import parseTemplateString from "../functions/templating";
import AceEditor from "../components/ace-editor";
import Request from "../components/collections/request";

const statusCategories = [
  {
    name: "informational",
    statusCodeRegex: new RegExp(/[1][0-9]+/),
    className: "info-response"
  },
  {
    name: "successful",
    statusCodeRegex: new RegExp(/[2][0-9]+/),
    className: "success-response"
  },
  {
    name: "redirection",
    statusCodeRegex: new RegExp(/[3][0-9]+/),
    className: "redir-response"
  },
  {
    name: "client error",
    statusCodeRegex: new RegExp(/[4][0-9]+/),
    className: "cl-error-response"
  },
  {
    name: "server error",
    statusCodeRegex: new RegExp(/[5][0-9]+/),
    className: "sv-error-response"
  },
  {
    // this object is a catch-all for when no other objects match and should always be last
    name: "unknown",
    statusCodeRegex: new RegExp(/.*/),
    className: "missing-data-response"
  }
];
const parseHeaders = xhr => {
  const headers = xhr
    .getAllResponseHeaders()
    .trim()
    .split(/[\r\n]+/);
  const headerMap = {};
  headers.forEach(line => {
    const parts = line.split(": ");
    const header = parts.shift().toLowerCase();
    const value = parts.join(": ");
    headerMap[header] = value;
  });
  return headerMap;
};
export const findStatusGroup = responseStatus =>
  statusCategories.find(status => status.statusCodeRegex.test(responseStatus));

export default {
  directives: {
    textareaAutoHeight
  },

  components: {
    Request,
    "pw-section": section,
    "pw-toggle": () => import("../components/toggle"),
    "pw-modal": () => import("../components/modal"),
    history: () => import("../components/history"),
    requestSection: () => import("../components/requestSection"),
    autocomplete: () => import("../components/autocomplete"),
    collections: () => import("../components/collections"),
    saveRequestAs: () => import("../components/collections/saveRequestAs"),
    ResponseBody: AceEditor
  },
  data() {
    return {
      showModal: false,
      showPreRequestScript: false,
      preRequestScript: "",
      copyButton: '<i class="material-icons">file_copy</i>',
      downloadButton: '<i class="material-icons">get_app</i>',
      doneButton: '<i class="material-icons">done</i>',
      isHidden: true,
      response: {
        status: "",
        headers: "",
        body: ""
      },
      previewEnabled: false,
      paramsWatchEnabled: true,
      expandResponse: false,

      /**
       * These are content types that can be automatically
       * serialized by postwoman.
       */
      knownContentTypes: [
        "application/json",
        "application/x-www-form-urlencoded"
      ],

      /**
       * These are a list of Content Types known to Postwoman.
       */
      validContentTypes: [
        "application/json",
        "application/hal+json",
        "application/xml",
        "application/x-www-form-urlencoded",
        "text/html",
        "text/plain"
      ],
      showRequestModal: false,
      editRequest: {},

      urlExcludes: {},
      responseBodyText: "",
      responseBodyType: "text",
      responseBodyMaxLines: 16
    };
  },
  watch: {
    urlExcludes: {
      deep: true,
      handler() {
        this.$store.commit("postwoman/applySetting", [
          "URL_EXCLUDES",
          Object.assign({}, this.urlExcludes)
        ]);
      }
    },
    contentType(val) {
      this.rawInput = !this.knownContentTypes.includes(val);
    },
    rawInput(status) {
      if (status && this.rawParams === "") this.rawParams = "{}";
      else this.setRouteQueryState();
    },
    "response.body": function(val) {
      if (
        this.response.body === "(waiting to send request)" ||
        this.response.body === "Loading..."
      ) {
        this.responseBodyText = this.response.body;
        this.responseBodyType = "text";
      } else {
        if (
          this.responseType === "application/json" ||
          this.responseType === "application/hal+json"
        ) {
          this.responseBodyText = JSON.stringify(this.response.body, null, 2);
          this.responseBodyType = "json";
        } else if (this.responseType === "text/html") {
          this.responseBodyText = this.response.body;
          this.responseBodyType = "html";
        } else {
          this.responseBodyText = this.response.body;
          this.responseBodyType = "text";
        }
      }
    },
    params: {
      handler: function(newValue) {
        if (!this.paramsWatchEnabled) {
          this.paramsWatchEnabled = true;
          return;
        }
        let path = this.path;
        let queryString = newValue
          .filter(({ key }) => !!key)
          .map(({ key, value }) => `${key}=${value}`)
          .join("&");
        queryString = queryString === "" ? "" : `?${queryString}`;
        if (path.includes("?")) {
          path = path.slice(0, path.indexOf("?")) + queryString;
        } else {
          path = path + queryString;
        }

        this.path = path;
      },
      deep: true
    },
    selectedRequest(newValue, oldValue) {
      // @TODO: Convert all variables to single request variable
      if (!newValue) return;
      this.url = newValue.url;
      this.path = newValue.path;
      this.method = newValue.method;
      this.auth = newValue.auth;
      this.httpUser = newValue.httpUser;
      this.httpPassword = newValue.httpPassword;
      this.passwordFieldType = newValue.passwordFieldType;
      this.bearerToken = newValue.bearerToken;
      this.headers = newValue.headers;
      this.params = newValue.params;
      this.bodyParams = newValue.bodyParams;
      this.rawParams = newValue.rawParams;
      this.rawInput = newValue.rawInput;
      this.contentType = newValue.contentType;
      this.requestType = newValue.requestType;
    },
    editingRequest(newValue) {
      this.editRequest = newValue;
      this.showRequestModal = true;
    }
  },
  computed: {
    url: {
      get() {
        return this.$store.state.request.url;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "url" });
      }
    },
    method: {
      get() {
        return this.$store.state.request.method;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "method" });
      }
    },
    path: {
      get() {
        return this.$store.state.request.path;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "path" });
      }
    },
    label: {
      get() {
        return this.$store.state.request.label;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "label" });
      }
    },
    auth: {
      get() {
        return this.$store.state.request.auth;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "auth" });
      }
    },
    httpUser: {
      get() {
        return this.$store.state.request.httpUser;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "httpUser" });
      }
    },
    httpPassword: {
      get() {
        return this.$store.state.request.httpPassword;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "httpPassword" });
      }
    },
    bearerToken: {
      get() {
        return this.$store.state.request.bearerToken;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "bearerToken" });
      }
    },
    headers: {
      get() {
        return this.$store.state.request.headers;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "headers" });
      }
    },
    params: {
      get() {
        return this.$store.state.request.params;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "params" });
      }
    },
    bodyParams: {
      get() {
        return this.$store.state.request.bodyParams;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "bodyParams" });
      }
    },
    rawParams: {
      get() {
        return this.$store.state.request.rawParams;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "rawParams" });
      }
    },
    rawInput: {
      get() {
        return this.$store.state.request.rawInput;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "rawInput" });
      }
    },
    requestType: {
      get() {
        return this.$store.state.request.requestType;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "requestType" });
      }
    },
    contentType: {
      get() {
        return this.$store.state.request.contentType;
      },
      set(value) {
        this.$store.commit("setState", { value, attribute: "contentType" });
      }
    },
    passwordFieldType: {
      get() {
        return this.$store.state.request.passwordFieldType;
      },
      set(value) {
        this.$store.commit("setState", {
          value,
          attribute: "passwordFieldType"
        });
      }
    },

    selectedRequest() {
      return this.$store.state.postwoman.selectedRequest;
    },
    editingRequest() {
      return this.$store.state.postwoman.editingRequest;
    },
    requestName() {
      return this.label;
    },
    statusCategory() {
      return findStatusGroup(this.response.status);
    },
    isValidURL() {
      if (this.showPreRequestScript) {
        // we cannot determine if a URL is valid because the full string is not known ahead of time
        return true;
      }
      const protocol = "^(https?:\\/\\/)?";
      const validIP = new RegExp(
        protocol +
          "(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5]).){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])$"
      );
      const validHostname = new RegExp(
        protocol +
          "(([a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]).)*([A-Za-z0-9]|[A-Za-z0-9][A-Za-z0-9-]*[A-Za-z0-9/])$"
      );
      return validIP.test(this.url) || validHostname.test(this.url);
    },
    hasRequestBody() {
      return ["POST", "PUT", "PATCH"].includes(this.method);
    },
    pathName() {
      return this.path.match(/^([^?]*)\??/)[1];
    },
    rawRequestBody() {
      const { bodyParams } = this;
      if (this.contentType === "application/json") {
        try {
          const obj = JSON.parse(
            `{${bodyParams
              .filter(({ key }) => !!key)
              .map(
                ({ key, value }) => `
              "${key}": "${value}"
              `
              )
              .join()}}`
          );
          return JSON.stringify(obj);
        } catch (ex) {
          return "invalid";
        }
      } else {
        return bodyParams
          .filter(({ key }) => !!key)
          .map(({ key, value }) => `${key}=${encodeURIComponent(value)}`)
          .join("&");
      }
    },
    headerString() {
      const result = this.headers
        .filter(({ key }) => !!key)
        .map(({ key, value }) => `${key}: ${value}`)
        .join(",\n");
      return result === "" ? "" : `${result}`;
    },
    queryString() {
      const result = this.params
        .filter(({ key }) => !!key)
        .map(({ key, value }) => `${key}=${encodeURIComponent(value)}`)
        .join("&");
      return result === "" ? "" : `?${result}`;
    },
    responseType() {
      return (this.response.headers["content-type"] || "")
        .split(";")[0]
        .toLowerCase();
    },
    requestCode() {
      if (this.requestType === "JavaScript XHR") {
        var requestString = [];
        requestString.push("const xhr = new XMLHttpRequest()");
        const user = this.auth === "Basic" ? "'" + this.httpUser + "'" : null;
        const pswd =
          this.auth === "Basic" ? "'" + this.httpPassword + "'" : null;
        requestString.push(
          "xhr.open('" +
            this.method +
            "', '" +
            this.url +
            this.pathName +
            this.queryString +
            "', true, " +
            user +
            ", " +
            pswd +
            ")"
        );
        if (this.auth === "Bearer Token") {
          requestString.push(
            "xhr.setRequestHeader('Authorization', 'Bearer " +
              this.bearerToken +
              "')"
          );
        }
        if (this.headers) {
          this.headers.forEach(function(element) {
            requestString.push(
              "xhr.setRequestHeader('" +
                element.key +
                "', '" +
                element.value +
                "')"
            );
          });
        }
        if (["POST", "PUT", "PATCH"].includes(this.method)) {
          const requestBody = this.rawInput
            ? this.rawParams
            : this.rawRequestBody;
          requestString.push(
            "xhr.setRequestHeader('Content-Length', " + requestBody.length + ")"
          );
          requestString.push(
            "xhr.setRequestHeader('Content-Type', '" +
              this.contentType +
              "; charset=utf-8')"
          );
          requestString.push("xhr.send(" + requestBody + ")");
        } else {
          requestString.push("xhr.send()");
        }
        return requestString.join("\n");
      } else if (this.requestType == "Fetch") {
        var requestString = [];
        var headers = [];
        requestString.push(
          'fetch("' + this.url + this.pathName + this.queryString + '", {\n'
        );
        requestString.push('  method: "' + this.method + '",\n');
        if (this.auth === "Basic") {
          var basic = this.httpUser + ":" + this.httpPassword;
          headers.push(
            '    "Authorization": "Basic ' +
              window.btoa(unescape(encodeURIComponent(basic))) +
              '",\n'
          );
        } else if (this.auth === "Bearer Token") {
          headers.push(
            '    "Authorization": "Bearer ' + this.bearerToken + '",\n'
          );
        }
        if (["POST", "PUT", "PATCH"].includes(this.method)) {
          const requestBody = this.rawInput
            ? this.rawParams
            : this.rawRequestBody;
          requestString.push("  body: " + requestBody + ",\n");
          headers.push('    "Content-Length": ' + requestBody.length + ",\n");
          headers.push(
            '    "Content-Type": "' + this.contentType + '; charset=utf-8",\n'
          );
        }
        if (this.headers) {
          this.headers.forEach(function(element) {
            headers.push(
              '    "' + element.key + '": "' + element.value + '",\n'
            );
          });
        }
        headers = headers.join("").slice(0, -2);
        requestString.push("  headers: {\n" + headers + "\n  },\n");
        requestString.push('  credentials: "same-origin"\n');
        requestString.push("}).then(function(response) {\n");
        requestString.push("  response.status\n");
        requestString.push("  response.statusText\n");
        requestString.push("  response.headers\n");
        requestString.push("  response.url\n\n");
        requestString.push("  return response.text()\n");
        requestString.push("}).catch(function(error) {\n");
        requestString.push("  error.message\n");
        requestString.push("})");
        return requestString.join("");
      } else if (this.requestType === "cURL") {
        var requestString = [];
        requestString.push("curl -X " + this.method + " \\\n");
        requestString.push(
          "  '" + this.url + this.pathName + this.queryString + "' \\\n"
        );
        if (this.auth === "Basic") {
          var basic = this.httpUser + ":" + this.httpPassword;
          requestString.push(
            "  -H 'Authorization: Basic " +
              window.btoa(unescape(encodeURIComponent(basic))) +
              "' \\\n"
          );
        } else if (this.auth === "Bearer Token") {
          requestString.push(
            "  -H 'Authorization: Bearer " + this.bearerToken + "' \\\n"
          );
        }
        if (this.headers) {
          this.headers.forEach(function(element) {
            requestString.push(
              "  -H '" + element.key + ": " + element.value + "' \\\n"
            );
          });
        }
        if (["POST", "PUT", "PATCH"].includes(this.method)) {
          const requestBody = this.rawInput
            ? this.rawParams
            : this.rawRequestBody;
          requestString.push(
            "  -H 'Content-Length: " + requestBody.length + "' \\\n"
          );
          requestString.push(
            "  -H 'Content-Type: " + this.contentType + "; charset=utf-8' \\\n"
          );
          requestString.push("  -d '" + requestBody + "' \\\n");
        }
        return requestString.join("").slice(0, -3);
      }
    }
  },
  methods: {
    handleUseHistory({
      label,
      method,
      url,
      path,
      usesScripts,
      preRequestScript
    }) {
      this.label = label;
      this.method = method;
      this.url = url;
      this.path = path;
      this.showPreRequestScript = usesScripts;
      this.preRequestScript = preRequestScript;
      // requestSection.scrollInto("request");
    },
    getVariablesFromPreRequestScript() {
      if (!this.preRequestScript) {
        return {};
      }
      return getEnvironmentVariablesFromScript(this.preRequestScript);
    },
    async makeRequest(auth, headers, requestBody, preRequestScript) {
      const requestOptions = {
        method: this.method,
        url: this.url + this.pathName + this.queryString,
        auth,
        headers,
        data: requestBody ? requestBody.toString() : null
      };
      if (preRequestScript) {
        const environmentVariables = getEnvironmentVariablesFromScript(
          preRequestScript
        );
        requestOptions.url = parseTemplateString(
          requestOptions.url,
          environmentVariables
        );
        requestOptions.data = parseTemplateString(
          requestOptions.data,
          environmentVariables
        );
        for (let k in requestOptions.headers) {
          const kParsed = parseTemplateString(k, environmentVariables);
          const valParsed = parseTemplateString(
            requestOptions.headers[k],
            environmentVariables
          );
          delete requestOptions.headers[k];
          requestOptions.headers[kParsed] = valParsed;
        }
      }
      if (typeof requestOptions.data === "string") {
        requestOptions.data = parseTemplateString(requestOptions.data);
      }
      const config = this.$store.state.postwoman.settings.PROXY_ENABLED
        ? {
            method: "POST",
            url: `https://postwoman.apollotv.xyz/`,
            data: requestOptions
          }
        : requestOptions;

      const response = await this.$axios(config);
      return this.$store.state.postwoman.settings.PROXY_ENABLED
        ? response.data
        : response;
    },
    async receiveRequest(requestSection) {
      this.$toast.clear();
      requestSection.scrollInto("response");
      // Start showing the loading bar as soon as possible.
      // The nuxt axios module will hide it when the request is made.
      this.$nuxt.$loading.start();

      if (requestSection.$refs.response.$el.classList.contains("hidden")) {
        requestSection.$refs.response.$el.classList.toggle("hidden");
      }
      requestSection.previewEnabled = false;
      requestSection.response.status = "Fetching...";
      requestSection.response.body = "Loading...";

      const auth =
        requestSection.auth === "Basic"
          ? {
              username: requestSection.httpUser,
              password: requestSection.httpPassword
            }
          : null;

      let headers = {};
      let headersObject = {};

      Object.keys(headers).forEach(id => {
        headersObject[headers[id].key] = headers[id].value;
      });
      headers = headersObject;

      // If the request has a body, we want to ensure Content-Length and
      // Content-Type are sent.
      let requestBody;
      if (requestSection.hasRequestBody) {
        requestBody = requestSection.rawInput ? requestSection.rawParams : requestSection.rawRequestBody;

        Object.assign(headers, {
          //'Content-Length': requestBody.length,
          "Content-Type": `${requestSection.contentType}; charset=utf-8`
        });
      }

      // If the request uses a token for auth, we want to make sure it's sent here.
      if (requestSection.auth === "Bearer Token")
        headers["Authorization"] = `Bearer ${requestSection.bearerToken}`;

      headers = Object.assign(
        // Clone the app headers object first, we don't want to
        // mutate it with the request headers added by default.
        Object.assign({}, requestSection.headers),

        // We make our temporary headers object the source so
        // that you can override the added headers if you
        // specify them.
        headers
      );

      Object.keys(headers).forEach(id => {
        headersObject[headers[id].key] = headers[id].value;
      });
      headers = headersObject;

      try {
        const startTime = Date.now();

        const payload = await this.makeRequest(
          auth,
          headers,
          requestBody,
          this.showPreRequestScript && this.preRequestScript
        );

        const duration = Date.now() - startTime;
        this.$toast.info(`Finished in ${duration}ms`, {
          icon: "done"
        });

        (() => {
          const status = (requestSection.response.status = payload.status);
          const headers = (requestSection.response.headers = payload.headers);

          // We don't need to bother parsing JSON, axios already handles it for us!
          const body = (requestSection.response.body = payload.data);

          const date = new Date().toLocaleDateString();
          const time = new Date().toLocaleTimeString();

          // Addition of an entry to the history component.
          const entry = {
            label: requestSection.requestName,
            status,
            date,
            time,
            method: requestSection.method,
            url: requestSection.url,
            path: requestSection.path,
            usesScripts: Boolean(requestSection.preRequestScript),
            preRequestScript: requestSection.preRequestScript,
            duration,
            star: false
          };
          this.$refs.historyComponent.addEntry(entry);
        })();
      } catch (error) {
        console.error(error);
        if (error.response) {
          requestSection.response.headers = error.response.headers;
          requestSection.response.status = error.response.status;
          requestSection.response.body = error.response.data;

          // Addition of an entry to the history component.
          const entry = {
            label: requestSection.requestName,
            status: requestSection.response.status,
            date: new Date().toLocaleDateString(),
            time: new Date().toLocaleTimeString(),
            method: requestSection.method,
            url: requestSection.url,
            path: requestSection.path,
            usesScripts: Boolean(requestSection.preRequestScript),
            preRequestScript: requestSection.preRequestScript
          };
          this.$refs.historyComponent.addEntry(entry);
          return;
        } else {
          this.response.status = error.message;
          this.response.body = error + ". Check console for details.";
          this.$toast.error(error + " (F12 for details)", {
            icon: "error"
          });
          if (!this.$store.state.postwoman.settings.PROXY_ENABLED) {
            this.$toast.info("Try enabling Proxy", {
              icon: "help",
              duration: 8000,
              action: {
                text: "Settings",
                onClick: (e, toastObject) => {
                  this.$router.push({ path: "/settings" });
                }
              }
            });
          }
        }
      }
    },
    getQueryStringFromPath() {
      let queryString,
        pathParsed = url.parse(this.path);
      return (queryString = pathParsed.query ? pathParsed.query : "");
    },
    queryStringToArray(queryString) {
      let queryParsed = querystring.parse(queryString);
      return Object.keys(queryParsed).map(key => ({
        key: key,
        value: queryParsed[key]
      }));
    },
    pathInputHandler() {
      let queryString = this.getQueryStringFromPath(),
        params = this.queryStringToArray(queryString);

      this.paramsWatchEnabled = false;
      this.params = params;
    },
    addRequestHeader() {
      this.$store.commit("addHeaders", {
        key: "",
        value: ""
      });
      return false;
    },
    removeRequestHeader(index) {
      this.$store.commit("removeHeaders", index);
      this.$toast.error("Deleted", {
        icon: "delete"
      });
    },
    addRequestParam() {
      this.$store.commit("addParams", { key: "", value: "" });
      return false;
    },
    removeRequestParam(index) {
      this.$store.commit("removeParams", index);
      this.$toast.error("Deleted", {
        icon: "delete"
      });
    },
    addRequestBodyParam() {
      this.$store.commit("addBodyParams", { key: "", value: "" });
      return false;
    },
    removeRequestBodyParam(index) {
      this.$store.commit("removeBodyParams", index);
      this.$toast.error("Deleted", {
        icon: "delete"
      });
    },
    formatRawParams(event) {
      if (event.which !== 13 && event.which !== 9) {
        return;
      }
      const textBody = event.target.value;
      const textBeforeCursor = textBody.substring(
        0,
        event.target.selectionStart
      );
      const textAfterCursor = textBody.substring(event.target.selectionEnd);
      if (event.which === 13) {
        event.preventDefault();
        const oldSelectionStart = event.target.selectionStart;
        const lastLine = textBeforeCursor.split("\n").slice(-1)[0];
        const rightPadding = lastLine.match(/([\s\t]*).*/)[1] || "";
        event.target.value =
          textBeforeCursor + "\n" + rightPadding + textAfterCursor;
        setTimeout(
          () =>
            (event.target.selectionStart = event.target.selectionEnd =
              oldSelectionStart + rightPadding.length + 1),
          1
        );
      } else if (event.which === 9) {
        event.preventDefault();
        const oldSelectionStart = event.target.selectionStart;
        event.target.value = textBeforeCursor + "\xa0\xa0" + textAfterCursor;
        event.target.selectionStart = event.target.selectionEnd =
          oldSelectionStart + 2;
        return false;
      }
    },
  },
  beforeDestroy() {
    document.removeEventListener("keydown", this._keyListener);
  }
};
</script>
