<template>
  <div class="edit-externals flex flex-col">
    <div v-if="!install || all.length > 1" class="select"
         :data-has-main="install ? '' : null">
      <dl v-for="([type, url, contents], i) of all" :key="i"
          class="flex"
          :class="{
            active: index === i,
            loading: install && i && !(url in install.deps),
            error: contents === false,
          }"
          @click="contents !== false && (index = i)">
        <dt v-text="type"/>
        <dd class="ellipsis flex-1">
          <a :href="url" target="_blank">&nearr;</a>
          <span v-text="decodeURIComponent(url)"/>
        </dd>
        <dd v-if="contents" v-text="formatLength(contents)" class="ml-2"/>
      </dl>
    </div>
    <div class="contents pos-rel flex-auto">
      <img v-if="img" :src="img">
      <vm-code
        class="abs-full"
        v-model="code"
        ref="code"
        readonly
        :cm-options="cmOptions"
        :mode="mode"
      />
    </div>
  </div>
</template>

<script>
import { string2uint8array } from '#/common';
import { objectEntries } from '#/common/object';
import VmCode from '#/common/ui/code';
import storage from '#/common/storage';

export default {
  props: ['value', 'cmOptions', 'commands', 'install', 'errors'],
  components: { VmCode },
  computed: {
    all() {
      const { code, deps = this.deps, url: mainUrl } = this.install || {};
      const { require = [], resources = {} } = this.value.meta || {};
      return [
        ...mainUrl ? [[this.i18n('editNavCode'), mainUrl, code]] : [],
        ...require.map(url => ['@require', url, deps[url]]),
        ...objectEntries(resources).map(([name, url]) => [`@resource ${name}`, url, deps[url]]),
      ];
    },
  },
  data() {
    return {
      code: null,
      deps: {},
      img: null,
      index: null,
      mode: null,
    };
  },
  watch: {
    async index(index) {
      const [type, url] = this.all[index] || [];
      if (!url) return;
      const { install } = this;
      const isMain = install && !index;
      const isReq = !isMain && type === '@require';
      let code;
      let contentType;
      let img;
      let raw;
      if (isMain) {
        code = install.code;
      } else {
        if (install) {
          raw = install.deps[url];
        } else {
          const key = this.value.custom.pathMap?.[url] || url;
          raw = await storage[isReq ? 'require' : 'cache'].getOne(key);
          if (!isReq) raw = storage.cache.makeDataUri(key, raw);
        }
        if (isReq || !raw) {
          code = raw;
        } else if (raw.startsWith('data:image')) {
          img = raw;
        } else {
          [contentType, code] = raw.split(',');
          if (code == null) { // workaround for bugs in old VM, see 2e135cf7
            const fileExt = url.match(/\.(\w+)([#&?]|$)/)?.[1] || '';
            contentType = /^(png|jpe?g|bmp|svgz?|gz|zip)$/i.test(fileExt)
              ? ''
              : `text/${fileExt.toLowerCase()}`;
            code = raw;
          }
          code = atob(code);
          if (/[\x80-\xFF]/.test(code)) {
            code = new TextDecoder().decode(string2uint8array(code));
          }
        }
      }
      this.img = img;
      this.mode = contentType === 'text/css' || /\.css([#&?]|$)/i.test(url) ? 'css' : null;
      this.code = code;
      this.$set(this.deps, url, code);
    },
    value() {
      this.$nextTick(() => {
        if (this.index >= this.all.length) this.index = 0;
      });
    },
  },
  async mounted() {
    this.index = 0;
  },
  methods: {
    formatLength(str) {
      const len = str?.length;
      return !len ? ''
        : len < 1024 && `${len} B`
        || len < 1024 * 1024 && `${len >> 10} k` // eslint-disable-line no-bitwise
        || `${len >> 20} M`; // eslint-disable-line no-bitwise
    },
  },
};
</script>

<style>
$outerPadX: 1rem;
$mainEntryBorder: 6px double;
.edit-externals {
  border-top: $mainEntryBorder var(--fill-8);
  > .select {
    min-height: 1.25rem;
    max-height: 15vh;
    overflow-y: auto;
    border-bottom: 2px solid var(--fill-3);
    padding-bottom: calc($outerPadX/2);
    &[data-has-main] dl:first-child {
      padding-top: .5em;
      padding-bottom: .5em;
      border-bottom: 1px solid var(--fill-3);
      position: sticky;
      top: 0;
      background: var(--fill-0);
    }
    dl {
      padding-right: $outerPadX;
      align-items: center;
      white-space: nowrap;
      &.active {
        font-weight: bold;
      }
      &.loading dd {
        color: var(--fill-7);
      }
      &.error dd {
        color: red;
      }
      &:not(.error) {
        cursor: pointer;
        &:hover dd {
          text-decoration: underline;
          a {
            text-decoration: none;
          }
        }
      }
    }
    dt {
      color: darkviolet;
      margin-left: $outerPadX;
      font-family: monospace;
    }
    a {
      padding: 0 .5em;
      cursor: alias;
      &:hover {
        background: var(--fill-3);
      }
    }
  }
  > .contents {
    > img {
      padding: 1rem;
      max-width: 100%;
      max-height: 100%;
      object-fit: contain;
    }
  }
  @media (prefers-color-scheme: dark) {
    .select {
      &.error dd {
        color: #ff4747;
      }
      dt {
        color: #c34ec3;
      }
    }
  }
}
</style>