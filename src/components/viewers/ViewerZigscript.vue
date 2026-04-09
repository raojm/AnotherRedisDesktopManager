<template>
  <div class="zigscript-viewer">
    <div class="zig-header" v-if="parsed">
      <span class="comp-name">{{ def ? def.name : 'Unknown' }}</span>
      <span class="comp-meta">id={{ parsed.header.id }} | ver={{ parsed.header.version }} | size={{ parsed.header.size }} | flag=0x{{ parsed.header.flag.toString(16) }}</span>
    </div>

    <JsonEditor v-if="parsed && !parsed.error" ref='editor' :content='displayContent' :readOnly='true' class='zig-editor'></JsonEditor>

    <div v-else-if="parsed && parsed.error" class="parse-error">
      <div class="error-title">Parse Error</div>
      <pre>{{ parsed.error }}</pre>
      <div class="raw-hex">Raw: {{ rawHexPreview }}</div>
    </div>

    <JsonEditor v-else ref='editor' :content='{error:"Waiting for data..."}' :readOnly='true'></JsonEditor>
  </div>
</template>

<script type="text/javascript">
import JsonEditor from '@/components/JsonEditor';

// 全局组件注册表（从 JSON 配置文件加载）
let CR = {};

export default {
  components: { JsonEditor },
  props: ['content'],
  data() {
    return {
      parsed: null,
      def: null,
    };
  },
  computed: {
    displayContent() {
      if (!this.parsed || this.parsed.error) return {};
      if (!this.def) {
        return {
          _header: {
            id: this.parsed.header.id,
            version: this.parsed.header.version,
            size: this.parsed.header.size,
            flag: '0x' + this.parsed.header.flag.toString(16)
          },
          _component: `unknown.${this.parsed.header.id}`,
          _raw: this.parsed.raw
        };
      }

      const result = {
        _header: {
          id: this.parsed.header.id,
          version: this.parsed.header.version,
          size: this.parsed.header.size,
          flag: '0x' + this.parsed.header.flag.toString(16)
        },
        _component: this.def.name
      };

      // 解析字段值
      let offset = 8;
      const view = this.parsed.view;

      for (const f of this.def.fields) {
        let val;
        switch (f.t) {
          case 'u8': val = view.getUint8(offset); offset += 1; break;
          case 'u16': val = view.getUint16(offset, true); offset += 2; break;
          case 'u32': val = view.getUint32(offset, true); offset += 4; break;
          case 'u64':
            const lo = view.getUint32(offset, true), hi = view.getUint32(offset + 4, true);
            val = BigInt.asUintN(64, (BigInt(hi) << 32n) | BigInt(lo));
            offset += 8;
            break;
          case 'i8': val = view.getInt8(offset); offset += 1; break;
          case 'i16': val = view.getInt16(offset, true); offset += 2; break;
          case 'i32': val = view.getInt32(offset, true); offset += 4; break;
          case 'f32': val = view.getFloat32(offset, true); offset += 4; break;
          case 'f64': val = view.getFloat64(offset, true); offset += 8; break;
          case 'string':
            const bytes = [];
            while (offset < view.byteLength) {
              const c = view.getUint8(offset);
              if (!c) break;
              bytes.push(c);
              offset++;
            }
            val = new TextDecoder().decode(new Uint8Array(bytes));
            break;
          default: val = null;
        }

        // 类型后缀
        const typeSuffix = ` :${f.t}`;
        if (typeof val === 'bigint') {
          result[f.n] = `${val.toString()} (0x${val.toString(16)})${typeSuffix}`;
        } else if (f.t === 'f32' || f.t === 'f64') {
          result[f.n] = `${val.toFixed(2)}${typeSuffix}`;
        } else {
          result[f.n] = `${val}${typeSuffix}`;
        }
      }

      return result;
    },
    rawHexPreview() {
      if (!this.content) return '';
      const buf = this.content instanceof Uint8Array ? this.content : this.unescapeBuffer(this.content);
      return Array.from(buf.slice(0, 32)).map(b => b.toString(16).padStart(2, '0')).join(' ');
    }
  },
  watch: {
    content: {
      immediate: true,
      handler(val) {
        if (!val) {
          this.parsed = null;
          this.def = null;
          return;
        }
        this.parsed = this.parseZigscript(val);
        this.def = this.parsed && !this.parsed.error ? CR[this.parsed.header.id] : null;
      }
    }
  },
  mounted() {
    // 尝试从 JSON 文件加载组件定义
    this.loadComponentDefinitions();
  },
  methods: {
    // 从 JSON 文件加载组件定义
    loadComponentDefinitions() {
      try {
        // 使用 Node.js fs 模块从固定配置目录读取
        const fs = require('fs');
        const path = require('path');
        const os = require('os');

        // 固定配置路径: ~/.config/zigscript/components.json
        const configPath = path.join(os.homedir(), '.config', 'zigscript', 'components.json');

        if (!fs.existsSync(configPath)) {
          console.warn('Component definitions not found at', configPath);
          console.warn('Please run: zig build export-components');
          return;
        }

        const data = JSON.parse(fs.readFileSync(configPath, 'utf8'));

        if (data && data.components && Array.isArray(data.components)) {
          // 转换 JSON 格式到内部格式
          CR = {};
          for (const comp of data.components) {
            CR[comp.id] = {
              name: comp.name,
              cat: comp.category,
              fields: comp.fields.map(f => ({ n: f.name, t: f.type }))
            };
          }
          console.log(`Loaded ${data.components.length} component definitions from ${configPath}`);

          // 如果已经解析了数据，重新获取定义
          if (this.parsed && !this.parsed.error) {
            this.def = CR[this.parsed.header.id] || null;
          }
        }
      } catch (err) {
        console.log('Failed to load component definitions JSON:', err);
      }
    },

    parseZigscript(content) {
      try {
        let buf;
        if (content instanceof Uint8Array) {
          buf = content;
        } else if (typeof content === 'string') {
          buf = this.unescapeBuffer(content);
        } else {
          return { error: 'Invalid content type' };
        }

        if (buf.length < 8) {
          return { error: 'Buffer too small (need at least 8 bytes for header)' };
        }

        const view = new DataView(buf.buffer, buf.byteOffset, buf.byteLength);

        return {
          header: {
            size: view.getUint16(0, true),
            version: view.getUint16(2, true),
            id: view.getUint16(4, true),
            flag: view.getUint16(6, true)
          },
          view: view,
          raw: Array.from(buf).map(b => b.toString(16).padStart(2, '0')).join(' ')
        };
      } catch (e) {
        return { error: e.message };
      }
    },

    unescapeBuffer(str) {
      const bytes = [];
      for (let i = 0; i < str.length; i++) {
        if (str[i] === '\\' && str[i + 1] === 'x') {
          bytes.push(parseInt(str.substr(i + 2, 2), 16));
          i += 3;
        } else {
          bytes.push(str.charCodeAt(i));
        }
      }
      return new Uint8Array(bytes);
    }
  }
};
</script>

<style scoped>
.zigscript-viewer { display: flex; flex-direction: column; height: 100%; }
.zig-header {
  background: linear-gradient(90deg, #1a1a2e 0%, #16213e 100%);
  padding: 14px 18px;
  border-bottom: 2px solid #0f3460;
  display: flex;
  align-items: center;
  gap: 14px;
  flex-shrink: 0;
}
.comp-cat {
  background: #e94560;
  color: #fff;
  padding: 6px 12px;
  border-radius: 6px;
  font-size: 13px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}
.comp-cat.cat-tag { background: #6c757d; }
.comp-cat.cat-core { background: #17a2b8; }
.comp-cat.cat-player { background: #28a745; }
.comp-cat.cat-auth { background: #fd7e14; }
.comp-cat.cat-position { background: #6f42c1; }
.comp-cat.cat-state { background: #20c997; }
.comp-cat.cat-request { background: #e83e8c; }
.comp-cat.cat-response { background: #007bff; }
.comp-cat.cat-room { background: #6610f2; }
.comp-cat.cat-game { background: #dc3545; }
.comp-cat.cat-system { background: #adb5bd; }
.comp-cat.cat-unknown { background: #6c757d; }
.comp-name {
  color: #fff;
  font-size: 16px;
  font-weight: 700;
}
.comp-meta {
  color: #a0d2eb;
  font-size: 13px;
  font-family: 'Monaco', 'Menlo', monospace;
  margin-left: auto;
}
.zig-editor { flex: 1; overflow: auto; }
.parse-error {
  padding: 20px;
  background: #1a1a2e;
  color: #e94560;
}
.error-title {
  font-size: 16px;
  font-weight: bold;
  margin-bottom: 10px;
}
.raw-hex {
  margin-top: 10px;
  padding: 10px;
  background: #0f0f1a;
  font-family: monospace;
  font-size: 12px;
  color: #a0d2eb;
}
</style>
