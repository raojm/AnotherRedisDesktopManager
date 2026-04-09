<template>
  <div>
    <!-- table toolbar -->
    <div>
      <!-- add button -->
      <el-button type="primary" @click="showEditDialog({})">{{ $t('message.add_new_line') }}</el-button>

      <!-- edit & add dialog -->
      <el-dialog :title="dialogTitle" :visible.sync="editDialog" @open="openDialog" :close-on-click-modal="false">
        <el-form label-position="top">

          <!-- if ttl support -->
          <el-form-item v-if="ttlSupport" label="Field">
            <el-row :gutter="10">
              <el-col :span="18">
                <InputBinary :content.sync="editLineItem.key" placeholder="Field"></InputBinary>
              </el-col>
              <el-col :span="6">
                <el-input v-model="editLineItem.ttl" placeholder="TTL (-1)" type="number"></el-input>
              </el-col>
            </el-row>
          </el-form-item>

          <!-- common field -->
          <el-form-item v-else label="Field">
            <InputBinary :content.sync="editLineItem.key" placeholder="Field"></InputBinary>
          </el-form-item>

          <el-form-item label="Value">
            <FormatViewer ref="formatViewer" :redisKey="redisKey" :dataMap="editLineItem" :content='editLineItem.value'></FormatViewer>
          </el-form-item>
        </el-form>

        <div slot="footer" class="dialog-footer">
          <el-button @click="editDialog = false">{{ $t('el.messagebox.cancel') }}</el-button>
          <el-button type="primary" @click="editLine">{{ $t('el.messagebox.confirm') }}</el-button>
        </div>
      </el-dialog>
    </div>

    <!-- vxe table must get a container with a fixed height -->
    <div class="content-table-container">
      <vxe-table
        ref="contentTable"
        size="mini" max-height="100%" min-height="72px"
        border="default" stripe show-overflow="title"
        :scroll-y="{enabled: true}"
        :row-config="{isHover: true, height: 34}"
        :column-config="{resizable: true}"
        :empty-text="$t('el.table.emptyText')"
        :data="hashData">
        <vxe-column type="seq" :title="'ID (Total: ' + total + ')'" width="150"></vxe-column>
        <vxe-column field="key" title="Key" sortable>
          <template v-slot="scope">
            {{ $util.bufToString(scope.row.key) }}
          </template>
        </vxe-column>
        <vxe-column field="value" title="Value" sortable>
          <template v-slot="scope">
            <div class="value-cell">
              <div class="value-preview">{{ getZigscriptPreview(scope.row.value) }}</div>
              <div class="value-raw">{{ $util.cutString($util.bufToString(scope.row.value), 80) }}</div>
            </div>
          </template>
        </vxe-column>
        <vxe-column v-if="ttlSupport" field="ttl" title="TTL" width="100" sortable></vxe-column>
        <vxe-column title="Operate" width="166">
          <template slot-scope="scope" slot="header">
            <el-input size="mini"
              :placeholder="$t('message.key_to_search')"
              :suffix-icon="loadingIcon"
              @keyup.native.enter='initShow()'
              v-model="filterValue">
            </el-input>
          </template>
          <template slot-scope="scope">
            <el-button type="text" @click="$util.copyToClipboard(scope.row.value)" icon="el-icon-document" :title="$t('message.copy')"></el-button>
            <el-button type="text" @click="showEditDialog(scope.row)" icon="el-icon-edit" :title="$t('message.edit_line')"></el-button>
            <el-button type="text" @click="deleteLine(scope.row)" icon="el-icon-delete" :title="$t('el.upload.delete')"></el-button>
            <el-button type="text" @click="dumpCommand(scope.row)" icon="fa fa-code" :title="$t('message.dump_to_clipboard')"></el-button>
          </template>
        </vxe-column>
      </vxe-table>
    </div>

    <!-- load more content -->
    <div class='content-more-container'>
      <el-button
        size='mini'
        @click='initShow(false)'
        :icon='loadingIcon'
        :disabled='loadMoreDisable'
        class='content-more-btn'>
        {{ $t('message.load_more_keys') }}
      </el-button>
    </div>
  </div>
</template>

<script>
import FormatViewer from '@/components/FormatViewer';
import InputBinary from '@/components/InputBinary';
import { VxeTable, VxeColumn } from 'vxe-table';
import versionCompare from 'node-version-compare';

export default {
  data() {
    return {
      total: 0,
      filterValue: '',
      editDialog: false,
      hashData: [], // {key: xxx, value: xxx}
      beforeEditItem: {},
      editLineItem: {},
      loadingIcon: '',
      pageSize: 200,
      searchPageSize: 2000,
      oneTimeListLength: 0,
      scanStream: null,
      loadMoreDisable: false,
      // 组件定义（会从 JSON 文件动态加载）
      componentDefs: null,
    };
  },
  components: {
    FormatViewer, InputBinary, VxeTable, VxeColumn,
  },
  props: ['client', 'redisKey'],
  computed: {
    dialogTitle() {
      return this.beforeEditItem.key ? this.$t('message.edit_line')
        : this.$t('message.add_new_line');
    },
    ttlSupport() {
      // avaiable since redis >= 7.4
      return versionCompare(this.client.ardmRedisVersion, '7.4') >= 0;
    },
  },
  watch: {
    hashData(newValue, oldValue) {
      // this.$refs.contentTable.refreshScroll()
      // scroll to bottom while loading more
      if (oldValue.length && (newValue.length > oldValue.length)) {
        setTimeout(() => {
          this.$refs.contentTable && this.$refs.contentTable.scrollTo(0, 99999999);
        }, 0);
      }
    }
  },
  methods: {
    initShow(resetTable = true) {
      resetTable && this.resetTable();
      this.loadingIcon = 'el-icon-loading';

      if (!this.scanStream) {
        this.initScanStream();
      } else {
        this.oneTimeListLength = 0;
        this.scanStream.resume();
      }

      // total lines
      this.initTotal();
    },
    initTotal() {
      this.client.hlen(this.redisKey).then((reply) => {
        this.total = reply;
      }).catch((e) => {});
    },
    resetTable() {
      // stop scanning first, #815
      this.scanStream && this.scanStream.pause();
      this.hashData = [];
      this.scanStream = null;
      this.oneTimeListLength = 0;
      this.loadMoreDisable = false;
    },
    initTTL(hashData, startIndex = 0) {
      if (!this.ttlSupport || !hashData.length) {
        return;
      }

      const keys = hashData.map(line => line.key);
      this.client.call('HTTL', this.redisKey, 'FIELDS', keys.length, ...keys).then(reply => {
        reply.forEach((ttl, index) => {
          this.hashData[startIndex + index].ttl = parseInt(ttl);
        });
      });
    },
    initScanStream() {
      const scanOption = { match: this.getScanMatch(), count: this.pageSize };
      scanOption.match != '*' && (scanOption.count = this.searchPageSize);

      this.scanStream = this.client.hscanBufferStream(
        this.redisKey,
        scanOption,
      );

      this.scanStream.on('data', (reply) => {
        const hashData = [];

        for (let i = 0; i < reply.length; i += 2) {
          hashData.push({
            key: reply[i],
            // keyDisplay: this.$util.bufToString(reply[i]),
            value: reply[i + 1],
            // valueDisplay: this.$util.bufToString(reply[i + 1]),
            ttl: -1
          });
        }

        const listLength = this.hashData.length;
        this.oneTimeListLength += hashData.length;
        this.hashData = this.hashData.concat(hashData);

        // init hash field ttls
        this.initTTL(hashData, listLength);

        if (this.oneTimeListLength >= this.pageSize) {
          this.scanStream.pause();
          this.loadingIcon = '';
        }
      });

      this.scanStream.on('end', () => {
        this.loadingIcon = '';
        this.loadMoreDisable = true;
      });

      this.scanStream.on('error', (e) => {
        this.loadingIcon = '';
        this.loadMoreDisable = true;
        this.$message.error(e.message);
      });
    },
    getScanMatch() {
      return this.filterValue ? `*${this.filterValue}*` : '*';
    },

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
          console.warn('KeyContentHash: Component definitions not found at', configPath);
          console.warn('Please run: zig build export-components');
          return;
        }

        const data = JSON.parse(fs.readFileSync(configPath, 'utf8'));

        if (data && data.components && Array.isArray(data.components)) {
          // 转换 JSON 格式到内部格式
          const defs = {};
          for (const comp of data.components) {
            defs[comp.id] = {
              name: comp.name,
              fields: comp.fields.map(f => ({ n: f.name, t: f.type }))
            };
          }
          this.componentDefs = defs;
          console.log(`KeyContentHash: Loaded ${data.components.length} component definitions from ${configPath}`);
        }
      } catch (err) {
        console.log('KeyContentHash: Failed to load component definitions JSON:', err);
      }
    },

    // 获取默认组件定义（内置）
    getDefaultComponentDefs() {
      return {
        10: { name: 'EntityUniqueID', fields: [{ n: 'unique_id', t: 'u64' }] },
        11: { name: 'OwnerEntityUniqueID', fields: [{ n: 'owner_unique_id', t: 'u64' }] },
        12: { name: 'EntityInitStatus', fields: [{ n: 'status', t: 'u8' }] },
        13: { name: 'ServerEntityTag', fields: [] },
        14: { name: 'GameRoomTag', fields: [] },
        15: { name: 'GameTableTag', fields: [] },
        16: { name: 'GameEntityTag', fields: [] },
        17: { name: 'LocalGameEntityTag', fields: [] },
        18: { name: 'RequestEntityTag', fields: [] },
        19: { name: 'LocalRequestTag', fields: [] },
        100: { name: 'Player', fields: [{ n: 'player_id', t: 'u64' }, { n: 'level', t: 'u32' }, { n: 'exp', t: 'u64' }, { n: 'nickname', t: 'string' }] },
        101: { name: 'PlayerProperty', fields: [{ n: 'hp', t: 'i32' }, { n: 'max_hp', t: 'i32' }, { n: 'mp', t: 'i32' }, { n: 'max_mp', t: 'i32' }, { n: 'attack', t: 'i32' }, { n: 'defense', t: 'i32' }, { n: 'is_dead', t: 'u8' }, { n: 'level', t: 'u32' }, { n: 'exp', t: 'u64' }] },
        102: { name: 'PlayerInfo', fields: [{ n: 'player_uin', t: 'u64' }, { n: 'game_id', t: 'u32' }, { n: 'gold', t: 'u32' }, { n: 'diamond', t: 'u32' }, { n: 'vip_level', t: 'u32' }] },
        200: { name: 'AuthenRequest', fields: [{ n: 'account', t: 'string' }, { n: 'password', t: 'string' }, { n: 'app_id', t: 'u32' }, { n: 'device_type', t: 'u8' }, { n: 'login_type', t: 'u8' }] },
        201: { name: 'AuthenResponse', fields: [{ n: 'result_code', t: 'i32' }, { n: 'player_uin', t: 'u64' }, { n: 'session_id', t: 'u64' }, { n: 'error_message', t: 'string' }] },
        300: { name: 'Position', fields: [{ n: 'x', t: 'f32' }, { n: 'y', t: 'f32' }, { n: 'z', t: 'f32' }] },
        301: { name: 'Velocity', fields: [{ n: 'vx', t: 'f32' }, { n: 'vy', t: 'f32' }, { n: 'vz', t: 'f32' }] },
        302: { name: 'Rotation', fields: [{ n: 'pitch', t: 'f32' }, { n: 'yaw', t: 'f32' }, { n: 'roll', t: 'f32' }] },
        400: { name: 'ClientState', fields: [{ n: 'is_background', t: 'u8' }, { n: 'afk_time_ms', t: 'u32' }] },
        401: { name: 'SkillState', fields: [{ n: 'skill_id', t: 'u32' }, { n: 'last_use_time', t: 'u64' }, { n: 'cooldown', t: 'u32' }, { n: 'target_id', t: 'u64' }] },
        402: { name: 'QuestState', fields: [{ n: 'quest_id', t: 'u32' }, { n: 'status', t: 'u8' }, { n: 'progress', t: 'u8' }] },
        403: { name: 'Inventory', fields: [{ n: 'item_id_1', t: 'u32' }, { n: 'count_1', t: 'u32' }, { n: 'item_id_2', t: 'u32' }, { n: 'count_2', t: 'u32' }, { n: 'item_id_3', t: 'u32' }, { n: 'count_3', t: 'u32' }, { n: 'gold', t: 'u64' }] },
        404: { name: 'CombatEffect', fields: [{ n: 'effect_type', t: 'u32' }, { n: 'effect_id', t: 'u32' }, { n: 'duration_ms', t: 'u32' }, { n: 'source_id', t: 'u64' }, { n: 'target_id', t: 'u64' }, { n: 'value', t: 'i32' }] },
        500: { name: 'MoveRequest', fields: [{ n: 'direction_x', t: 'f32' }, { n: 'direction_y', t: 'f32' }, { n: 'speed', t: 'f32' }] },
        501: { name: 'QuestRequest', fields: [{ n: 'quest_id', t: 'u32' }, { n: 'action_type', t: 'u8' }] },
        502: { name: 'CombatRequest', fields: [{ n: 'skill_id', t: 'u32' }, { n: 'target_id', t: 'u64' }] },
        503: { name: 'ReviveRequest', fields: [{ n: 'revive_type', t: 'u8' }] },
        504: { name: 'UseItemRequest', fields: [{ n: 'item_id', t: 'u32' }, { n: 'slot', t: 'u8' }] },
        505: { name: 'LoginRequest', fields: [{ n: 'lobby_version', t: 'u32' }, { n: 'account', t: 'string' }, { n: 'role_uin', t: 'u64' }, { n: 'role_name', t: 'string' }] },
        506: { name: 'LoginResponse', fields: [{ n: 'result_id', t: 'i32' }, { n: 'role_uin', t: 'u64' }, { n: 'role_name', t: 'string' }] },
        507: { name: 'EnterRoomRequest', fields: [{ n: 'room_id', t: 'i32' }] },
        508: { name: 'EnterRoomResponse', fields: [{ n: 'result_id', t: 'i32' }, { n: 'room_id', t: 'i32' }] },
        509: { name: 'SitDownRequest', fields: [{ n: 'room_id', t: 'i32' }, { n: 'table_id', t: 'i32' }, { n: 'seat_id', t: 'i8' }, { n: 'sit_down_type', t: 'i8' }, { n: 'table_name', t: 'string' }, { n: 'table_passwd', t: 'string' }, { n: 'game_map_id', t: 'i32' }, { n: 'game_mod', t: 'i32' }] },
        510: { name: 'SitDownResponse', fields: [{ n: 'result_id', t: 'i32' }, { n: 'room_id', t: 'i32' }, { n: 'table_id', t: 'i32' }, { n: 'seat_id', t: 'i8' }] },
        511: { name: 'StandUpRequest', fields: [{ n: 'room_id', t: 'i32' }, { n: 'table_id', t: 'i32' }, { n: 'mode', t: 'i32' }] },
        512: { name: 'StandUpResponse', fields: [{ n: 'result_id', t: 'i32' }] },
        513: { name: 'LeaveRoomRequest', fields: [{ n: 'room_id', t: 'i32' }] },
        514: { name: 'LeaveRoomResponse', fields: [{ n: 'result_id', t: 'i32' }] },
        516: { name: 'GameMoveRequest', fields: [{ n: 'player_uin', t: 'u64' }, { n: 'target_x', t: 'f32' }, { n: 'target_y', t: 'f32' }, { n: 'target_z', t: 'f32' }, { n: 'velocity_x', t: 'f32' }, { n: 'velocity_y', t: 'f32' }, { n: 'velocity_z', t: 'f32' }] },
        517: { name: 'GameMoveResponse', fields: [{ n: 'result_code', t: 'i32' }, { n: 'current_x', t: 'f32' }, { n: 'current_y', t: 'f32' }, { n: 'current_z', t: 'f32' }] },
        600: { name: 'RoomComponent', fields: [{ n: 'room_id', t: 'u32' }, { n: 'room_type', t: 'u8' }, { n: 'player_count', t: 'u8' }, { n: 'max_players', t: 'u8' }, { n: 'status', t: 'u8' }] },
        601: { name: 'SeatComponent', fields: [{ n: 'seat_index', t: 'u8' }, { n: 'is_ready', t: 'u8' }, { n: 'player_uin', t: 'u64' }] },
        700: { name: 'KeepAliveRequest', fields: [{ n: 'player_uin', t: 'u64' }, { n: 'timestamp', t: 'u64' }] },
        701: { name: 'KeepAliveResponse', fields: [{ n: 'result_code', t: 'i32' }, { n: 'server_time', t: 'u64' }] }
      };
    },

    getZigscriptPreview(buf) {
      // 尝试解析 ZigScript 组件，返回预览字符串
      if (!buf || buf.length < 8) return '';

      try {
        const view = new DataView(buf.buffer, buf.byteOffset, buf.byteLength);
        const id = view.getUint16(4, true);

        // 使用动态加载的组件定义，如果没有则使用默认的
        const defs = this.componentDefs || this.getDefaultComponentDefs();

        const def = defs[id];
        if (!def) return `[Unknown(${id})]`;

        // Tag 组件只显示名称
        if (!def.fields || def.fields.length === 0) {
          return `[${def.name}]`;
        }

        // 解析字段值
        let o = 8;
        const parts = [`[${def.name}]`];
        for (const f of def.fields.slice(0, 4)) { // 最多显示4个字段
          let val;
          switch (f.t) {
            case 'u8': val = view.getUint8(o); o += 1; break;
            case 'u16': val = view.getUint16(o, true); o += 2; break;
            case 'u32': val = view.getUint32(o, true); o += 4; break;
            case 'u64':
              const lo = view.getUint32(o, true), hi = view.getUint32(o + 4, true);
              val = `0x${(BigInt(hi) << 32n | BigInt(lo)).toString(16).slice(-8)}`;
              o += 8;
              break;
            case 'i8': val = view.getInt8(o); o += 1; break;
            case 'i16': val = view.getInt16(o, true); o += 2; break;
            case 'i32': val = view.getInt32(o, true); o += 4; break;
            case 'f32': val = view.getFloat32(o, true).toFixed(2); o += 4; break;
            case 'f64': val = view.getFloat64(o, true).toFixed(2); o += 8; break;
            case 'string':
              const bytes = [];
              while (o < buf.length) {
                const c = view.getUint8(o);
                if (!c) break;
                bytes.push(c);
                o++;
              }
              val = new TextDecoder().decode(new Uint8Array(bytes));
              if (val.length > 10) val = val.slice(0, 10) + '...';
              break;
            default: val = '?';
          }
          parts.push(`${f.n}=${val}`);
        }
        if (def.fields.length > 4) parts.push('...');
        return parts.join(' | ');
      } catch (e) {
        return '';
      }
    },
    openDialog() {
      this.$nextTick(() => {
        this.$refs.formatViewer.autoFormat();
      });
    },
    showEditDialog(row) {
      this.editLineItem = this.$util.cloneObjWithBuff(row);
      this.beforeEditItem = row;
      this.editDialog = true;
    },
    dumpCommand(item) {
      const lines = item ? [item] : this.hashData;
      const params = lines.map(line => `${this.$util.bufToQuotation(line.key)} ${
        this.$util.bufToQuotation(line.value)}`);

      const command = `HSET ${this.$util.bufToQuotation(this.redisKey)} ${params.join(' ')}`;
      this.$util.copyToClipboard(command);
      this.$message.success({ message: this.$t('message.copy_success'), duration: 800 });
    },
    editLine() {
      const key = this.redisKey;
      const { client } = this;
      const before = this.beforeEditItem;

      const afterKey = this.editLineItem.key;
      const afterValue = this.$refs.formatViewer.getContent();
      const afterTTL = parseInt(this.editLineItem.ttl);

      if (!afterKey || !afterValue) {
        return;
      }

      this.editDialog = false;

      client.hset(
        key,
        afterKey,
        afterValue,
      ).then((reply) => {
        // edit key && key changed
        if (before.key && !before.key.equals(afterKey)) {
          client.hdel(key, before.key);
        }

        // set ttl if supportted
        if (this.ttlSupport && afterTTL > 0) {
          this.client.call('HEXPIRE', key, afterTTL, "FIELDS", 1, afterKey);
        }

        // this.initShow(); // do not reinit, #786
        const newLine = Object.assign(
          {}, before,
          { key: afterKey, value: afterValue, ttl: afterTTL > 0 ? afterTTL : -1}
        );

        // edit line
        if (before.key) {
          this.$set(this.hashData, this.hashData.indexOf(before), newLine);
        }
        // new line
        else {
          this.hashData.push(newLine);
          this.total++;
        }

        // reply==1:new field; reply==0 field exists
        this.$message.success({
          message: reply == 1 ? this.$t('message.add_success') : this.$t('message.modify_success'),
          duration: 1000,
        });
      }).catch((e) => { this.$message.error(e.message); });
    },
    deleteLine(row) {
      this.$confirm(
        this.$t('message.confirm_to_delete_row_data'),
        { type: 'warning' },
      ).then(() => {
        this.client.hdel(
          this.redisKey,
          row.key,
        ).then((reply) => {
          if (reply == 1) {
            this.$message.success({
              message: this.$t('message.delete_success'),
              duration: 1000,
            });

            // this.initShow(); // do not reinit, #786
            this.hashData.splice(this.hashData.indexOf(row), 1);
            this.total--;
          }
        }).catch((e) => { this.$message.error(e.message); });
      }).catch(() => {});
    },
  },
  mounted() {
    this.initShow();
    this.loadComponentDefinitions();
  },
};
</script>

<style scoped>
.value-cell {
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.value-preview {
  color: #58a6ff;
  font-weight: 600;
  font-size: 12px;
}
.value-raw {
  color: #8b949e;
  font-size: 11px;
  font-family: 'Monaco', 'Menlo', monospace;
}
</style>
