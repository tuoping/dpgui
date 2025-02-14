<template>
  <!-- list -->
  <div class="w-100" v-if="Array.isArray(jdata)">
    <v-list>
      <v-list-item v-for="item in jdata" :key="item.name">
        <v-list-item-content>
          <DargsItem :jdata="item" ref="subitem" />
        </v-list-item-content>
      </v-list-item>
    </v-list>
  </div>
  <div class="w-100 mx-1" v-else-if="jdata.object == 'Argument'">
    <v-row class="w-100">
      <v-col cols="auto">
        <v-checkbox v-model="check" v-if="jdata.optional"></v-checkbox> </v-col
      ><v-col>
        <v-row v-if="jdata.type.length > 1">
          <v-col cols="auto">
            <v-subheader
              >{{ $t('message.typeof', { name: jdata.name }) }}
              <small v-if="jdata.optional">{{ $t('message.optional') }}</small></v-subheader
            >
          </v-col>
          <v-col>
            <v-select
              :items="jdata.type"
              v-model="select_type"
              :hint="$t('message.select_type', {name: jdata.name})"
            ></v-select>
          </v-col>
        </v-row>
        <!-- text areas for list -->
        <v-textarea
          v-if="select_type == 'list'"
          v-model="value"
          :hint="$t('message.one_element_per_row') + '\n' + jdata.doc"
          :placeholder="select_type"
          :rules="!jdata.optional ? rules.required : []"
          clearable
          @input="up"
        >
          <template v-slot:label>
            <div>
              {{ jdata.name }} <small v-if="jdata.optional">{{ $t('message.optional') }}</small>
            </div>
          </template>
          <template v-slot:message="{ message, key }">
            <div v-html="message.replaceAll('\n', '<br/>')" :key="key"></div>
          </template>
        </v-textarea>
        <!-- text field for str, int, float -->
        <v-text-field
          :hint="jdata.doc"
          v-model="value"
          :placeholder="select_type"
          :rules="[
            ...(!jdata.optional ? rules.required : []),
            ...(['int', 'float'].includes(select_type) ? rules.number : []),
          ]"
          v-else-if="['str', 'int', 'float'].includes(select_type)"
          clearable
          @input="up"
        >
          <template v-slot:label>
            <div>
              {{ jdata.name }} <small v-if="jdata.optional">{{ $t('message.optional') }}</small>
            </div>
          </template>
          <template v-slot:message="{ message, key }">
            <div v-html="message.replaceAll('\n', '<br/>')" :key="key"></div>
          </template>
        </v-text-field>

        <!-- switch for bool -->
        <v-switch
          v-else-if="select_type == 'bool'"
          v-model="value"
          :hint="jdata.doc"
          :disabled="!check"
          persistent-hint
        >
          <template v-slot:label>
            <div>
              {{ jdata.name }} <small v-if="jdata.optional">{{ $t('message.optional') }}</small>
            </div>
          </template>
          <template v-slot:message="{ message, key }">
            <div
              v-html="message.replaceAll('\n', '<br/>')"
              :key="key"
            ></div> </template
        ></v-switch>

        <div v-else-if="select_type == 'dict'">
          <!-- dict sub_fields -->
          <v-list v-if="Object.keys(jdata.sub_fields).length">
            <v-list-item-title> {{ jdata.name }}</v-list-item-title>
            <v-list-item-subtitle> {{ jdata.doc }}</v-list-item-subtitle>
            <v-list-item v-for="item in jdata.sub_fields" :key="item.name">
              <!-- jdata.sub_fields is object -->
              <v-list-item-content>
                <DargsItem :jdata="item" ref="subitem" />
              </v-list-item-content>
            </v-list-item>
          </v-list>

          <!-- dict variant -->
          <v-list v-if="Object.keys(jdata.sub_variants).length" subheader>
            <v-list-item-title> {{ jdata.name }}</v-list-item-title>
            <v-list-item-subtitle> {{ jdata.doc }}</v-list-item-subtitle>
            <v-list-item v-for="item in jdata.sub_variants" :key="item.name">
              <!-- jdata.sub_fields is object -->
              <v-list-item-content>
                <DargsItem :jdata="item" ref="subvariant" />
              </v-list-item-content>
            </v-list-item>
          </v-list>
        </div>
      </v-col>
    </v-row>
  </div>
  <div class="w-100" v-else-if="jdata.object == 'Variant'">
    <v-tabs v-model="tab" show-arrows>
      <v-tab v-for="item in jdata.choice_dict" :key="item.name">
        {{ item.name }}
      </v-tab>
    </v-tabs>
    <v-tabs-items v-model="tab">
      <!-- important: use eager prop to let this.$refs["subitem"][this.tab] not undefined -->
      <v-tab-item v-for="item in jdata.choice_dict" :key="item.name" eager>
        <v-card flat>
          <DargsItem :jdata="item" ref="subitem" />
        </v-card>
      </v-tab-item>
    </v-tabs-items>
  </div>
</template>

<script>
export default {
  name: "DargsItem",
  props: {
    jdata: [Object, Array],
  },
  data() {
    var tab = 0;
    if (this.jdata.object == "Variant") {
      if (this.jdata.default_tag) {
        tab = Object.keys(this.jdata.choice_dict).indexOf(
          this.jdata.default_tag
        );
      }
      if (tab < 0) tab = 0;
    }
    return {
      tab: tab,
      value: null,
      check: !this.jdata.optional,
      rules: {
        required: [(value) => !!value || "Required."],
        number: [(value) => !value || !isNaN(value) || "Number required."],
      },
      select_type: this.jdata.type && this.jdata.type[0],
    };
  },
  methods: {
    /**
     * Get a object.
     * @returns {object} The returned obj.
     */
    dvalue: function () {
      if (Array.isArray(this.jdata)) {
        return Object.fromEntries(
          new Map(
            this.$refs["subitem"]
              .filter((vv) => !vv.jdata.optional || vv.check)
              .map((vv) => [vv.jdata.name, vv.dvalue()])
          )
        );
      } else if (this.jdata.object == "Argument") {
        if (this.select_type == "list") {
          // textarea -> list
          if (!this.value) return [];
          return this.value
            .trim()
            .split("\n")
            .map((v) => {
              try {
                return JSON.parse(v);
              } catch (e) {
                return v;
              }
            });
        } else if (["str", "int", "float"].includes(this.select_type)) {
          if (!this.value) {
            if (this.select_type == "str") return "";
            else return 0;
          }
          if (!this.select_type == "str") {
            return Number.parseFloat(this.value);
          }
          return this.value;
        } else if (this.select_type == "bool") {
          return this.value;
        } else if (this.select_type == "NoneType") {
          return null;
        } else if (this.select_type == "dict") {
          const sub_fields = Object.keys(this.jdata.sub_fields).length
            ? Object.fromEntries(
                new Map(
                  this.$refs["subitem"]
                    .filter((vv) => !vv.jdata.optional || vv.check)
                    .map((vv) => [vv.jdata.name, vv.dvalue()])
                )
              )
            : Object();

          const sub_variants = Object.keys(this.jdata.sub_variants).length
            ? Object.assign(
                ...this.$refs["subvariant"].map((vv) => vv.dvalue())
              )
            : Object();
          return {
            ...sub_fields,
            ...sub_variants,
          };
        }
      } else if (this.jdata.object == "Variant") {
        // the current vaiant: this.tab
        const flag = Object.fromEntries(
          new Map([
            [
              this.jdata.flag_name,
              Object.keys(this.jdata.choice_dict)[this.tab],
            ],
          ])
        );
        return {
          ...flag,
          ...this.$refs["subitem"][this.tab].dvalue(),
        };
      }
      return null;
    },
    up: function () {
      if (this.value) {
        this.check = true;
      }
    },
    /**
     * Load data from an object.
     * @param {object} obj the object to load
     */
    load: function (obj) {
      const load_subitem = () => {
        if (this.$refs["subitem"])
          this.$refs["subitem"].forEach((vv) => {
            // check if it exists, name and alias
            if (vv.jdata.name in obj) {
              // exists
              vv.load(obj[vv.jdata.name]);
            }
            vv.jdata.alias.forEach((aa) => {
              if (aa in obj) vv.load(obj[aa]);
            });
            if (vv.jdata.optional) {
              vv.check = vv.jdata.name in obj;
            }
          });
      };
      if (Array.isArray(this.jdata)) {
        // array
        load_subitem();
      } else if (this.jdata.object == "Argument") {
        if (this.jdata.type.includes("list") && Array.isArray(obj)) {
          // list -> multiple line str
          this.select_type = "list";
          this.value = obj.map(JSON.stringify).join("\n");
        } else if (this.jdata.type.includes("str") && typeof obj == "string") {
          this.select_type = "str";
          this.value = obj;
        } else if (
          ["int", "float"].some((tt) => this.jdata.type.includes(tt)) &&
          typeof obj == "number"
        ) {
          // str, int, float -> str
          this.value = String(obj);
          if (this.jdata.type.includes("float")) {
            this.select_type = "float";
          }
          else if (this.jdata.type.includes("int")) {
            this.select_type = "int";
          }
        } else if (
          this.jdata.type.includes("bool") &&
          typeof obj == "boolean"
        ) {
          // bool
          this.value = obj;
          this.select_type = "bool";
        } else if (this.jdata.type.includes("NoneType") && obj === null) {
          this.value = obj;
          this.select_type = "NoneType";
        } else if (this.jdata.type.includes("dict")) {
          this.select_type = "dict";
          load_subitem();
          if (this.$refs["subvariant"])
            this.$refs["subvariant"].forEach((vv) => {
              vv.load(obj);
            });
        }
      } else if (this.jdata.object == "Variant") {
        var tab = Object.keys(this.jdata.choice_dict).indexOf(
          obj[this.jdata.flag_name] || this.jdata.default_tag
        );
        // loop alias
        if (tab < 0)
          this.$refs["subitem"].some((vv, index) => {
            if (
              vv.jdata.alias.includes(
                obj[this.jdata.flag_name] || this.jdata.default_tag
              )
            ) {
              tab = index;
              return true;
            }
            return false;
          });
        this.tab = tab;
        this.$refs["subitem"][this.tab].load(obj);
      }
    },
  },
};
</script>

<style scoped>
.v-tab {
  text-transform: none !important;
}
.w-100 {
  width: 100%;
}
.mx-1 {
  margin-left: 1em;
}
</style>
