<template>
  <div class="zigscript-viewer">
    <div class="zig-header" v-if="parsed">
      <span class="comp-cat" :class="'cat-' + (def ? def.cat : 'unknown')">{{ def ? def.cat : 'unknown' }}</span>
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

const CR = {
  10:{name:'EntityUniqueID',cat:'core',fields:[{n:'unique_id',t:'u64'}]},
  11:{name:'OwnerEntityUniqueID',cat:'core',fields:[{n:'owner_unique_id',t:'u64'}]},
  12:{name:'EntityInitStatus',cat:'core',fields:[{n:'status',t:'u8'}]},
  13:{name:'ServerEntityTag',cat:'tag',fields:[]},
  14:{name:'GameRoomTag',cat:'tag',fields:[]},
  15:{name:'GameTableTag',cat:'tag',fields:[]},
  16:{name:'GameEntityTag',cat:'tag',fields:[]},
  17:{name:'LocalGameEntityTag',cat:'tag',fields:[]},
  18:{name:'RequestEntityTag',cat:'tag',fields:[]},
  19:{name:'LocalRequestTag',cat:'tag',fields:[]},
  100:{name:'Player',cat:'player',fields:[{n:'player_id',t:'u64'},{n:'level',t:'u32'},{n:'exp',t:'u64'},{n:'nickname',t:'string'}]},
  101:{name:'PlayerProperty',cat:'player',fields:[
    {n:'hp',t:'i32'},{n:'max_hp',t:'i32'},{n:'mp',t:'i32'},{n:'max_mp',t:'i32'},
    {n:'attack',t:'i32'},{n:'defense',t:'i32'},{n:'is_dead',t:'u8'},{n:'level',t:'u32'},{n:'exp',t:'u64'}
  ]},
  102:{name:'PlayerInfo',cat:'player',fields:[{n:'player_uin',t:'u64'},{n:'game_id',t:'u32'},{n:'gold',t:'u32'},{n:'diamond',t:'u32'},{n:'vip_level',t:'u32'}]},
  200:{name:'AuthenRequest',cat:'auth',fields:[{n:'account',t:'string'},{n:'password',t:'string'},{n:'app_id',t:'u32'},{n:'device_type',t:'u8'},{n:'login_type',t:'u8'}]},
  201:{name:'AuthenResponse',cat:'auth',fields:[{n:'result_code',t:'i32'},{n:'player_uin',t:'u64'},{n:'session_id',t:'u64'},{n:'error_message',t:'string'}]},
  300:{name:'Position',cat:'position',fields:[{n:'x',t:'f32'},{n:'y',t:'f32'},{n:'z',t:'f32'}]},
  301:{name:'Velocity',cat:'position',fields:[{n:'vx',t:'f32'},{n:'vy',t:'f32'},{n:'vz',t:'f32'}]},
  302:{name:'Rotation',cat:'position',fields:[{n:'pitch',t:'f32'},{n:'yaw',t:'f32'},{n:'roll',t:'f32'}]},
  400:{name:'ClientState',cat:'state',fields:[{n:'is_background',t:'u8'},{n:'afk_time_ms',t:'u32'}]},
  401:{name:'SkillState',cat:'state',fields:[{n:'skill_id',t:'u32'},{n:'last_use_time',t:'u64'},{n:'cooldown',t:'u32'},{n:'target_id',t:'u64'}]},
  402:{name:'QuestState',cat:'state',fields:[{n:'quest_id',t:'u32'},{n:'status',t:'u8'},{n:'progress',t:'u8'}]},
  403:{name:'Inventory',cat:'state',fields:[
    {n:'item_id_1',t:'u32'},{n:'count_1',t:'u32'},{n:'item_id_2',t:'u32'},{n:'count_2',t:'u32'},
    {n:'item_id_3',t:'u32'},{n:'count_3',t:'u32'},{n:'gold',t:'u64'}
  ]},
  404:{name:'CombatEffect',cat:'state',fields:[
    {n:'effect_type',t:'u32'},{n:'effect_id',t:'u32'},{n:'duration_ms',t:'u32'},
    {n:'source_id',t:'u64'},{n:'target_id',t:'u64'},{n:'value',t:'i32'}
  ]},
  500:{name:'MoveRequest',cat:'request',fields:[{n:'direction_x',t:'f32'},{n:'direction_y',t:'f32'},{n:'speed',t:'f32'}]},
  501:{name:'QuestRequest',cat:'request',fields:[{n:'quest_id',t:'u32'},{n:'action_type',t:'u8'}]},
  502:{name:'CombatRequest',cat:'request',fields:[{n:'skill_id',t:'u32'},{n:'target_id',t:'u64'}]},
  503:{name:'ReviveRequest',cat:'request',fields:[{n:'revive_type',t:'u8'}]},
  504:{name:'UseItemRequest',cat:'request',fields:[{n:'item_id',t:'u32'},{n:'slot',t:'u8'}]},
  505:{name:'LoginRequest',cat:'room',fields:[{n:'lobby_version',t:'u32'},{n:'account',t:'string'},{n:'role_uin',t:'u64'},{n:'role_name',t:'string'}]},
  506:{name:'LoginResponse',cat:'room',fields:[{n:'result_id',t:'i32'},{n:'role_uin',t:'u64'},{n:'role_name',t:'string'}]},
  507:{name:'EnterRoomRequest',cat:'room',fields:[{n:'room_id',t:'i32'}]},
  508:{name:'EnterRoomResponse',cat:'room',fields:[{n:'result_id',t:'i32'},{n:'room_id',t:'i32'}]},
  509:{name:'SitDownRequest',cat:'room',fields:[
    {n:'room_id',t:'i32'},{n:'table_id',t:'i32'},{n:'seat_id',t:'i8'},{n:'sit_down_type',t:'i8'},
    {n:'table_name',t:'string'},{n:'table_passwd',t:'string'},{n:'game_map_id',t:'i32'},{n:'game_mod',t:'i32'}
  ]},
  510:{name:'SitDownResponse',cat:'room',fields:[{n:'result_id',t:'i32'},{n:'room_id',t:'i32'},{n:'table_id',t:'i32'},{n:'seat_id',t:'i8'}]},
  511:{name:'StandUpRequest',cat:'room',fields:[{n:'room_id',t:'i32'},{n:'table_id',t:'i32'},{n:'mode',t:'i32'}]},
  512:{name:'StandUpResponse',cat:'room',fields:[{n:'result_id',t:'i32'}]},
  513:{name:'LeaveRoomRequest',cat:'room',fields:[{n:'room_id',t:'i32'}]},
  514:{name:'LeaveRoomResponse',cat:'room',fields:[{n:'result_id',t:'i32'}]},
  516:{name:'GameMoveRequest',cat:'game',fields:[
    {n:'player_uin',t:'u64'},{n:'target_x',t:'f32'},{n:'target_y',t:'f32'},{n:'target_z',t:'f32'},
    {n:'velocity_x',t:'f32'},{n:'velocity_y',t:'f32'},{n:'velocity_z',t:'f32'}
  ]},
  517:{name:'GameMoveResponse',cat:'game',fields:[{n:'result_code',t:'i32'},{n:'current_x',t:'f32'},{n:'current_y',t:'f32'},{n:'current_z',t:'f32'}]},
  600:{name:'RoomComponent',cat:'room',fields:[{n:'room_id',t:'u32'},{n:'room_type',t:'u8'},{n:'player_count',t:'u8'},{n:'max_players',t:'u8'},{n:'status',t:'u8'}]},
  601:{name:'SeatComponent',cat:'room',fields:[{n:'seat_index',t:'u8'},{n:'is_ready',t:'u8'},{n:'player_uin',t:'u64'}]},
  700:{name:'KeepAliveRequest',cat:'system',fields:[{n:'player_uin',t:'u64'},{n:'timestamp',t:'u64'}]},
  701:{name:'KeepAliveResponse',cat:'system',fields:[{n:'result_code',t:'i32'},{n:'server_time',t:'u64'}]}
};

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
            size: this.parsed.header.size,
            version: this.parsed.header.version,
            id: this.parsed.header.id,
            flag: `0x${this.parsed.header.flag.toString(16)}`
          },
          _note: 'Unknown component ID, no field definitions available'
        };
      }
      if (!this.def.fields || !this.def.fields.length) {
        return { _note: 'Tag component (no data fields)' };
      }

      const result = { _component: `${this.def.cat}.${this.def.name}` };
      for (const f of this.def.fields) {
        const val = this.parsed.data[f.n];
        const formattedVal = val !== undefined ? fmt(val, f.t) : '<undefined>';
        result[f.n] = `${formattedVal} :${f.t}`;
      }
      return result;
    },
    rawHexPreview() {
      if (!this.content) return '';
      const buf = Buffer.isBuffer(this.content)
        ? this.content
        : Buffer.from(this.content, 'binary');
      const len = Math.min(buf.length, 80);
      return Array.from(buf.slice(0, len)).map(b => b.toString(16).padStart(2,'0')).join(' ') + (buf.length > len ? '...' : '');
    },
  },
  watch: {
    content: {
      immediate: true,
      handler() {
        this.parse();
      },
    },
  },
  methods: {
    parse() {
      try {
        let buf = this.content;

        // 支持字符串输入（转义格式）
        if (typeof buf === 'string') {
          const unescaped = buf
            .replace(/\\x([0-9a-fA-F]{2})/g, (_, hex) => String.fromCharCode(parseInt(hex, 16)))
            .replace(/\\n/g, '\n')
            .replace(/\\r/g, '\r')
            .replace(/\\t/g, '\t')
            .replace(/\\\\/g, '\\');
          buf = Buffer.from(unescaped, 'binary');
        }

        if (!Buffer.isBuffer(buf) || buf.length < 8) {
          this.parsed = { error: `Invalid buffer: need >= 8 bytes, got ${buf ? buf.length : 0}`, header: {} };
          this.def = null;
          return;
        }

        const view = new DataView(buf.buffer, buf.byteOffset, buf.byteLength);

        const header = {
          size: view.getUint16(0, true),
          version: view.getUint16(2, true),
          id: view.getUint16(4, true),
          flag: view.getUint16(6, true),
        };

        this.def = CR[header.id] || null;

        let data = {};
        if (this.def && this.def.fields && this.def.fields.length > 0) {
          data = parseFields(view, buf, this.def.fields);
        }

        this.parsed = { header, data, error: null };
      } catch (e) {
        this.parsed = { error: e.message || String(e), header: {} };
        this.def = null;
      }
    },

    getContent() {
      return this.content;
    },

    copyContent() {
      if (!this.parsed) return this.content.toString();
      if (this.parsed.error) return JSON.stringify({ error: this.parsed.error }, null, 2);
      return JSON.stringify(this.displayContent, null, 2);
    },
  },
};

function parseFields(view, buf, fields) {
  const r = {};
  let o = 8;
  for (const f of fields) {
    let val; switch(f.t){
      case'u8':val=view.getUint8(o);o+=1;break;
      case'u16':val=view.getUint16(o,true);o+=2;break;
      case'u32':val=view.getUint32(o,true);o+=4;break;
      case'u64':
        const lo=view.getUint32(o,true),hi=view.getUint32(o+4,true);o+=8;
        val=`0x${(BigInt(hi)<<32n|BigInt(lo)).toString(16).toUpperCase().padStart(16,'0')}`;
        break;
      case'i8':val=view.getInt8(o);o+=1;break;
      case'i16':val=view.getInt16(o,true);o+=2;break;
      case'i32':val=view.getInt32(o,true);o+=4;break;
      case'f32':val=view.getFloat32(o,true);o+=4;break;
      case'f64':val=view.getFloat64(o,true);o+=8;break;
      case'string':
        const b=[];while(o<buf.length){const c=view.getUint8(o);if(!c)break;b.push(c);o++}
        val=new TextDecoder().decode(new Uint8Array(b));break;
      default:val=null;
    }r[f.n]=val;
  }return r;
}

function fmt(val,type){
  if(val===null||val===undefined)return'<null>';
  if(typeof val==='number'){
    if(type&&type.startsWith('f'))return Number(val).toFixed(4);
    return `${val} (0x${Number(val).toString(16).toUpperCase()})`;
  }
  if(typeof val==='string')return `"${val}"`;
  return String(val);
}
</script>

<style scoped>
.zigscript-viewer { height: 100%; display: flex; flex-direction: column; }
.zig-header {
  display: flex; align-items: center; gap: 12px;
  padding: 12px 16px; background: #1e1e1e;
  border-bottom: 1px solid #333; flex-shrink: 0;
}
.comp-name { font-size: 16px; font-weight: 700; color: #fff; }
.comp-cat {
  font-size: 11px; padding: 4px 10px; border-radius: 4px;
  text-transform: uppercase; font-weight: 700; letter-spacing: 0.5px; color: white;
}
.comp-meta {
  font-size: 13px; color: #58a6ff; margin-left: auto;
  font-family: 'Monaco', 'Menlo', monospace;
  background: #0d1117;
  padding: 6px 12px;
  border-radius: 6px;
  border: 1px solid #30363d;
}
.cat-core { background: #da3633; }
.cat-tag { background: #6e7681; }
.cat-player { background: #1f6feb; }
.cat-auth { background: #8957e5; }
.cat-position { background: #238636; }
.cat-state { background: #bf8700; }
.cat-request { background: #bd561d; }
.cat-room { background: #2ea043; }
.cat-game { background: #6e7681; }
.cat-system { background: #6e7681; }
.cat-unknown { background: #484f58; }

.zig-editor { flex: 1; overflow: hidden; }

.parse-error {
  padding: 20px; text-align: center;
}
.error-title { color: #f85149; font-size: 16px; font-weight: 600; margin-bottom: 10px; }
.parse-error pre {
  background: #3d1217; color: #f85149; padding: 12px; border-radius: 6px;
  text-align: left; font-family: monospace; font-size: 12px; white-space: pre-wrap;
  max-height: 200px; overflow: auto;
}
.raw-hex {
  margin-top: 12px; padding: 8px; background: #21262d; border-radius: 4px;
  font-family: monospace; font-size: 11px; color: #6e7681; word-break: break-all;
}
</style>
