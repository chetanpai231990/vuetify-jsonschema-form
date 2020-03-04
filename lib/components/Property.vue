<template>
  <v-form  ref="form">
    <!-- Hide const ? Or make a readonly field -->
    <div v-if="fullSchema && fullSchema.const === undefined && fullSchema['x-display'] !== 'hidden'" class="vjsf-property">
      
      <!-- Datetime picker -->
      <v-layout v-if="schema.show_as === 'timestamp'" row>
        <v-flex fluid>
          <v-text-field
            v-if="disabled"
            :value="modelWrapper[modelKey]"
            :label="label"
            :name="fullKey"
            :hint="schema.description"
            :required="required"
            :rules="rules"
            prepend-icon="event"
            readonly
            
          />
          <v-datetime-picker
            v-else
            :datetime="modelWrapper[modelKey]"
            v-model="modelWrapper[modelKey]"
            :label="label"
            prepend-icon="event"
            @change="change"
            @input="dateTimeChanged"
            @showAsChanged="e => $emit('typechange', e)"
            :readonly="readonly"
          />
          <tooltip slot="append-outer" :html-description="htmlDescription" />
        </v-flex>
      </v-layout>
      
      <!-- Date picker -->
      <v-menu v-else-if="fullSchema.show_as === 'string' && schema.format === 'date'" ref="menu" v-model="menu" :close-on-content-click="false"
              :nudge-right="40"
              :return-value.sync="modelWrapper[modelKey]"
              :disabled="disabled"
              transition="scale-transition"
              offset-y
              full-width
              min-width="290px"
      >
        <template v-slot:activator="{on}">
          <v-text-field
            v-model="modelWrapper[modelKey]"
            :label="label"
            :name="fullKey"
            :required="required"
            :rules="rules"
            :clearable="!required"
            prepend-icon="event"
            readonly
            v-on="on"
          >
            <tooltip slot="append-outer" :html-description="htmlDescription" />
          </v-text-field>
        </template>
        <v-date-picker v-model="modelWrapper[modelKey]" no-title scrollable>
          <v-spacer />
          <v-btn text class="v-btn--flat" :style="oldFlat" @click="menu = false">
            Cancel
          </v-btn>
          <v-btn text class="v-btn--flat primary--text" @click="$refs.menu.save(modelWrapper[modelKey]); change(); input()">
            OK
          </v-btn>
        </v-date-picker>
      </v-menu>

      <!-- Color picking -->
      <template v-else-if="fullSchema.format === 'hexcolor'">
        <template v-if="fullSchema['x-display'] === 'color-picker'">
          <v-input
            :name="fullKey"
            :label="label"
            :required="required"
            :rules="rules"
            :disabled="disabled"
          >
            <tooltip slot="append" :html-description="htmlDescription" />
            &nbsp;&nbsp;
            <v-menu :close-on-content-click="false" :close-on-click="true" direction="bottom" offset-y>
              <template v-slot:activator="{on}">
                <div :style="`background-color: ${modelWrapper[modelKey]};`" :class="modelWrapper[modelKey] ? 'color-picker-trigger' : 'color-picker-trigger color-picker-trigger-empty'" v-on="on" />
              </template>
              <color-picker :value="modelWrapper[modelKey]" :preset-colors="options.colors.swatches" @input="(val) => {modelWrapper[modelKey] = val.hex; input(); change()}" />
            </v-menu>
          </v-input>
        </template>
        <v-input v-else
                :name="fullKey"
                :label="label"
                :required="required"
                :rules="rules"
                :disabled="disabled"
        >
          <tooltip slot="append" :html-description="htmlDescription" />
          &nbsp;&nbsp;
          <swatches
            v-model="modelWrapper[modelKey]"
            :disabled="disabled"
            :colors="options.colors"
            :trigger-style="{width:'36px', height:'36px'}"
            shapes="circles"
            @input="input();change()"
          />
        </v-input>
      </template>

      <!-- Select field based on an enum (array or simple value) -->
      <template v-else-if="(fullSchema.show_as === 'array' && fullSchema.items.enum) || fullSchema.enum">
        <div class="d-inline-flex">
          <v-switch v-if="fullSchema.optional != null && fullSchema.optional === true && !readonly" v-model="optionalSwitch" @change="switchChanged()"  color="green"/>
          <v-select
            v-model="modelWrapper[modelKey]"
            style="width:90%"
            :items="selectItems"
            :name="fullKey"
            :label="label"
            :required="required"
            :rules="rules"
            :disabled="disabled"
            :readonly="readonly"
            :multiple="fullSchema.show_as === 'array'"
            @change="change;optionalSwitch = true;"
            @input="input"
          >
            <!-- <template slot="prepend" v-if="fullSchema.optional != null && fullSchema.optional === true && !readonly" >
              <v-switch v-model="optionalSwitch" @change="switchChanged()"  color="green"/>
            </template> -->
            <template slot="selection" slot-scope="data">
              <div class="v-select__selection v-select__selection--comma">
                <select-icon v-if="itemIcon" :value="data.item" />
                <span v-if="![null, undefined].includes(data.item)">{{ data.item + (fullSchema.show_as === 'array' && data.index !== modelWrapper[modelKey].length - 1 ? ',&nbsp;' : '') }}</span>
              </div>
            </template>
            <template slot="item" slot-scope="data">
              <select-icon v-if="itemIcon" :value="data.item" />
              <select-item :title="data.item" :options="options" />
            </template>
            <tooltip slot="append-outer" :html-description="htmlDescription" />
          </v-select>
        </div>
      </template>

      <!-- Select field based on a oneOf on a simple type (array or simple value) -->
      <!-- cf https://github.com/mozilla-services/react-jsonfullSchema-form/issues/532 -->
      <template v-else-if="oneOfSelect">
        <v-select
          v-model="modelWrapper[modelKey]"
          :items="selectItems"
          :name="fullKey"
          :label="label"
          :required="required"
          :disabled="disabled"
          :rules="rules"
          :clearable="!required"
          :multiple="fullSchema.show_as === 'array'"
          :item-text="itemTitle"
          :item-value="itemKey"
          @change="change"
          @input="input"
        >
          <template slot="selection" slot-scope="data">
            <div class="v-select__selection v-select__selection--comma">
              <select-icon v-if="itemIcon" :value="data.item[itemIcon]" />
              <span v-if="![null, undefined].includes(data.item[itemTitle])">{{ data.item[itemTitle] + (fullSchema.show_as === 'array' && data.index !== modelWrapper[modelKey].length - 1 ? ',&nbsp;' : '') }}</span>
            </div>
          </template>
          <template slot="item" slot-scope="data">
            <select-icon v-if="itemIcon" :value="data.item[itemIcon]" />
            <select-item :title="data.item[itemTitle]" :options="options" />
          </template>
          <tooltip slot="append-outer" :html-description="htmlDescription" />
        </v-select>
      </template>

      <!-- Select field on an ajax response or from an array in another part of the data -->
      <v-select v-else-if="fullSchema['x-display'] !== 'list' && (fromUrl || fullSchema['x-fromData'])"
                v-model="modelWrapper[modelKey]"
                :items="selectItems"
                :name="fullKey"
                :label="label"
                :no-data-text="options.noDataMessage"
                :disabled="disabled"
                :required="required"
                :rules="rules"
                :item-text="itemTitle"
                :item-value="itemKey"
                :return-object="(fullSchema.show_as === 'array' && fullSchema.items.show_as === 'object') || fullSchema.show_as === 'object'"
                :clearable="!required"
                :loading="loading"
                :multiple="fullSchema.show_as === 'array'"
                @change="change"
                @input="input"
      >
        <template slot="selection" slot-scope="data">
          <div class="v-select__selection v-select__selection--comma">
            <select-icon v-if="itemIcon" :value="data.item[itemIcon]" />
            <span v-if="![null, undefined].includes(data.item[itemTitle])">{{ data.item[itemTitle] + (fullSchema.show_as === 'array' && data.index !== modelWrapper[modelKey].length - 1 ? ',&nbsp;' : '') }}</span>
          </div>
        </template>
        <template slot="item" slot-scope="data">
          <select-icon v-if="itemIcon" :value="data.item[itemIcon]" />
          <select-item :title="data.item[itemTitle]" :options="options" />
        </template>
        <tooltip slot="append-outer" :html-description="htmlDescription" />
      </v-select>

      <!-- auto-complete field on an ajax response with query -->
      <v-autocomplete v-else-if="fromUrlWithQuery"
                      v-model="modelWrapper[modelKey]"
                      :items="selectItems"
                      :search-input.sync="q"
                      :name="fullKey"
                      :label="label"
                      :no-data-text="options.noDataMessage"
                      :disabled="disabled"
                      :required="required"
                      :rules="rules"
                      :item-text="itemTitle"
                      :item-value="itemKey"
                      :return-object="(fullSchema.show_as === 'array' && fullSchema.items.show_as === 'object') || fullSchema.show_as === 'object'"
                      :clearable="!required"
                      :filter="() => true"
                      :placeholder="options.searchMessage"
                      :loading="loading"
                      :multiple="fullSchema.show_as === 'array'"
                      @change="change"
                      @input="input"
      >
        <template slot="selection" slot-scope="data">
          <select-icon v-if="itemIcon" :value="data.item[itemIcon]" />
          <div v-if="![null, undefined].includes(data.item[itemTitle])">
            {{ data.item[itemTitle] + (fullSchema.show_as === 'array' && data.index !== modelWrapper[modelKey].length - 1 ? ',&nbsp;' : '') }}
          </div>
        </template>
        <template slot="item" slot-scope="data">
          <select-icon v-if="itemIcon" :value="data.item[itemIcon]" />
          <select-item :title="data.item[itemTitle]" :options="options" />
        </template>
        <tooltip slot="append-outer" :html-description="htmlDescription" />
      </v-autocomplete>

      <!-- Long text field in a textarea -->
      <v-textarea
        v-else-if="fullSchema.show_as === 'string' && (fullSchema.maxLength && fullSchema.maxLength > 1000 && fullSchema['x-display'] !== 'single-line')"
        v-model="modelWrapper[modelKey]"
        :name="fullKey"
        :label="label"
        :disabled="disabled"
        :required="required"
        :rules="rules"
        filled
        class="v-text-field--box v-text-field--enclosed"
        @change="change"
        @input="input"
      >
        <tooltip slot="append-outer" :html-description="htmlDescription" />
      </v-textarea>

      <!-- text field displayed as password -->
      <v-text-field v-else-if="fullSchema.show_as === 'string' && fullSchema['x-display'] === 'password'"
                    v-model="modelWrapper[modelKey]"
                    :name="fullKey"
                    :label="label"
                    :disabled="disabled"
                    :required="required"
                    :rules="rules"
                    type="password"
                    @change="change"
                    @input="input"
      >
        <tooltip slot="append-outer" :html-description="htmlDescription" />
      </v-text-field>

      <!-- Handle Null Type :  -->
      <v-text-field v-else-if="fullSchema.show_as === 'null'"
                    style="width:90%"
                    value="null"
                    :name="fullKey"
                    :label="label"
                    :disabled="disabled"
                    :required="required"
                    :rules="rules"
                    :readonly="true"
                    @change="change"
                    @input="input"
      >
        <tooltip slot="append-outer" :html-description="htmlDescription" />
      </v-text-field>

      <!-- Simple text field -->
      <v-textarea v-else-if="fullSchema.show_as === 'string' || fullSchema.show_as === 'utf8string' || fullSchema.show_as === 'numericstring'"
                    style="width:90%"
                    :rows="modelWrapper[modelKey].trim().split('\n').length"
                    auto-grow
                    :value="modelWrapper[modelKey]"
                    :name="fullKey"
                    :label="label"
                    :disabled="disabled"
                    :required="required"
                    :rules="rules"
                    :readonly="readonly"
                    @change="modelWrapper[modelKey] = $event"
                    :counter="!readonly ? fullSchema.maximum:false"
                    dense
      >
        <tooltip slot="append-outer" :html-description="htmlDescription" />
      </v-textarea>


      <!-- Simple number fields -->
      <div v-else-if="fullSchema.show_as === 'number' || fullSchema.show_as === 'integer'">
          <!-- <v-subheader v-if="readonly" 
                  style="height:25px !important;color:white"> 
                  {{ label }} : {{ modelWrapper[modelKey] }}
          </v-subheader> -->
          <v-text-field 
                  style="width:90%"
                  :value="modelWrapper[modelKey]"
                  :name="fullKey"
                  :label="label"
                  :min="fullSchema.minimum"
                  :max="fullSchema.maximum"
                  :step="fullSchema.show_as === 'integer' ? 1 : 0.01"
                  :disabled="disabled"
                  :required="required"
                  :rules="rules"
                  :readonly="readonly"
                  type="number"
                  @change="modelWrapper[modelKey] = Number($event)"
                  @input="optionalSwitch =true; input"
                  @focus="dialog=false"
                  dense
          >
            <tooltip slot="append-outer" :html-description="htmlDescription" />
            <template v-if="fullSchema.optional != null && fullSchema.optional === true && !readonly" v-slot:prepend>
              <v-switch v-model="optionalSwitch" @change="switchChanged()" style="margin-top: 0px; !important;padding-top: 0px !important;" color="green"/>
            </template>
            <template v-slot:append-outer>
              <v-menu v-if="!readonly" transition="slide-x-transition" bottom left :close-on-content-click='dialog'>
                <template v-slot:activator="{ on }">
                  <span v-on="on">
                    <v-icon dark left style="color: #35495e;cursor:pointer;margin-top:5px">settings_applications</v-icon>
                  </span>
                </template>
                <v-card-text @click.stop="" style="background-color: whitesmoke" >
                  <v-container grid-list-md>
                    <v-layout wrap>
                      <v-flex xs20 sm6 md4>
                        <v-text-field label="Name" :value="label" readonly></v-text-field>
                      </v-flex>
                      <v-flex xs12 sm6 md4>
                        <v-text-field  label="Current Type" readonly :value="fullSchema.type"></v-text-field>
                      </v-flex>
                      <v-flex xs12 sm6 md4>
                        <v-select
                          v-model="selectedShowAsItem"
                          :items="showAsItems"
                          label="Show As"
                        ></v-select>
                      </v-flex>
                      <v-flex xs12 sm6 md12>
                        <v-textarea 
                          :rows="1" 
                          auto-grow 
                          label="Default" 
                          placeholder="Set Default Value" 
                          v-model="defaultValueforType" 
                          clearable
                          clear-icon="cancel">
                        </v-textarea>
                      </v-flex>
                    </v-layout>
                  </v-container>
                </v-card-text>
                <v-card-actions style="background-color: whitesmoke">
                  <v-spacer></v-spacer>
                  <v-btn color="blue darken-1" flat >Cancel</v-btn>
                  <!-- <v-btn color="blue darken-1" flat @click="dialog=true;reset(label)">Reset</v-btn> -->
                  <v-btn color="blue darken-1" flat @click="dialog=true;typechange({name:modelKey,type:fullSchema.type,show_as:selectedShowAsItem, default: defaultValueforType})">Save</v-btn>
                </v-card-actions> 
              </v-menu>
            </template>
          </v-text-field>
      </div>
      
      <!-- IP Address fields -->
      <vue-ip v-else-if="fullSchema.show_as === 'ip'"
            :label="label"
            :ip="modelWrapper[modelKey]"
            :index="index"
            :on-change="ipChange"
            :placeholder="true"
            :readonly="readonly"
            @showAsChanged="e => $emit('typechange', e)"
          >
            {{ label }}
      </vue-ip>

      <!-- Coordinates fields -->
      <v-text-field v-else-if="fullSchema.show_as === 'coordinates'"
                    style="width:90%"
                    :value="modelWrapper[modelKey] | convertCoordinates"
                    :name="fullKey"
                    :label="label"
                    :min="fullSchema.minimum"
                    :max="fullSchema.maximum"
                    :disabled="disabled"
                    :readonly="readonly"
                    :required="required"
                    :rules="rules"
                    @input="coordinatesToInteger($event)"
                    dense
      >
        <tooltip slot="append-outer" :html-description="htmlDescription" />
      </v-text-field>

      <!-- Octet String fields as Normal Strings-->
      <v-textarea v-else-if="fullSchema.show_as === 'hex'"
                    :rows="(modelWrapper[modelKey].length /60 )"
                    auto-grow
                    style="width:90%"
                    :value="modelWrapper[modelKey]"
                    :label="label"
                    @change="modelWrapper[modelKey]= $event"
                    :name="fullKey"
                    :min="fullSchema.minimum"
                    :max="fullSchema.maximum"
                    :disabled="disabled"
                    :required="required"
                    :readonly="readonly"
                    :rules="!readonly ? rules: false"
                    :counter="!readonly ? fullSchema.maximum * 2 : false"
                    dense
      >
        <template v-slot:label>
          <div>
            {{ label +' (' + fullSchema.show_as + ')'}}
          </div>
        </template>
        <template v-slot:append-outer>
          <v-menu v-if="!readonly" transition="slide-x-transition" bottom left :close-on-content-click='dialog'>
            <template v-slot:activator="{ on }">
              <span v-on="on">
                <v-icon dark left style="color: #35495e;cursor:pointer;margin-top:5px">settings_applications</v-icon>
              </span>
            </template>
            <v-card-text @click.stop="" style="background-color: whitesmoke" >
              <v-container grid-list-md>
                <v-layout wrap>
                  <v-flex xs20 sm6 md4>
                    <v-text-field label="Name" :value="label" readonly></v-text-field>
                  </v-flex>
                  <v-flex xs12 sm6 md4>
                    <v-text-field  label="Current Type" readonly :value="fullSchema.type"></v-text-field>
                  </v-flex>
                  <v-flex xs12 sm6 md4>
                    <v-select
                      v-model="selectedShowAsItem"
                      :items="showAsItemsforString"
                      label="Show As"
                    ></v-select>
                  </v-flex>
                  <v-flex xs12 sm6 md12>
                    <v-textarea 
                      :rows="1" 
                      clearable
                      clear-icon="cancel"
                      placeholder="Set Default Value" 
                      auto-grow label="Default" 
                      v-model="defaultValueforType">
                    </v-textarea>
                  </v-flex>
                </v-layout>
              </v-container>
            </v-card-text>
            <v-card-actions style="background-color: whitesmoke">
              <v-spacer></v-spacer>
              <v-btn color="blue darken-1" flat >Cancel</v-btn>
              <!-- <v-btn color="blue darken-1" flat @click="dialog=true;reset(label)">Reset</v-btn> -->
              <v-btn color="blue darken-1" flat @click="dialog=true;typechange({name:label,type:fullSchema.type,show_as:selectedShowAsItem, default: defaultValueforType})">Save</v-btn>
            </v-card-actions> 
          </v-menu>
        </template>
        <tooltip slot="append-outer" :html-description="htmlDescription" />
      </v-textarea>

      <!-- Octet String fields as Type :: HEX -->
      <div v-else-if="fullSchema.show_as === 'octet string'">
        <!-- <v-subheader v-if="readonly" 
                  style="height:25px !important;color:white"> 
                  {{ label }} : {{ modelWrapper[modelKey] | convertHextoAscii}}
        </v-subheader> -->
        <v-textarea 
                      :rows="modelWrapper[modelKey].trim().toUpperCase().split('0A').length === 1 ? (modelWrapper[modelKey].length / 60 ) : modelWrapper[modelKey].trim().toUpperCase().split('0A').length"
                      auto-grow
                      style="width:90%"   
                      :value="modelWrapper[modelKey] | convertHextoAscii"
                      :name="fullKey"
                      :min="fullSchema.minimum"
                      :max="fullSchema.maximum"
                      :disabled="disabled"
                      :readonly="readonly"
                      :required="required"
                      :rules="!readonly ? rules: false"
                      @change="octetStringChanged($event)"
                      :counter="!readonly ? fullSchema.maximum : false"
                      dense
        >
          <template v-slot:label>
            <div>
              {{ label +' (' + 'ascii' + ')'}}
            </div>
          </template>
          <tooltip slot="append-outer" :html-description="htmlDescription" />
          <template v-if="fullSchema.optional != null && fullSchema.optional === true && !readonly" v-slot:prepend>
            <v-switch v-model="optionalSwitch" @change="switchChanged()" style="margin-top: 0px; !important;padding-top: 0px !important;" color="green"/>
          </template>
          <template v-slot:append-outer>
            <v-menu v-if="!readonly" transition="slide-x-transition" bottom left :close-on-content-click='dialog'>
              <template v-slot:activator="{ on }">
                <span v-on="on">
                  <v-icon dark left style="color: #35495e;cursor:pointer;margin-top:5px">settings_applications</v-icon>
                </span>
              </template>
              <v-card-text @click.stop="" style="background-color: whitesmoke" >
                <v-container grid-list-md>
                  <v-layout wrap>
                    <v-flex xs20 sm6 md4>
                      <v-text-field label="Name" :value="label" readonly></v-text-field>
                    </v-flex>
                    <v-flex xs12 sm6 md4>
                      <v-text-field  label="Current Type" readonly :value="fullSchema.type"></v-text-field>
                    </v-flex>
                    <v-flex xs12 sm6 md4>
                      <v-select
                        v-model="selectedShowAsItem"
                        :items="showAsItemsforString"
                        label="Show As"
                      ></v-select>
                    </v-flex>
                    <v-flex xs12 sm6 md12>
                      <v-textarea 
                        :rows="1" 
                        clearable
                        clear-icon="cancel"
                        placeholder="Set Default Value" 
                        auto-grow label="Default" 
                        v-model="defaultValueforType">
                      </v-textarea>
                    </v-flex>
                  </v-layout>
                </v-container>
              </v-card-text>
              <v-card-actions style="background-color: whitesmoke">
                <v-spacer></v-spacer>
                <v-btn color="blue darken-1" flat >Cancel</v-btn>
                <!-- <v-btn color="blue darken-1" flat @click="dialog=true;reset(label)">Reset</v-btn> -->
                <v-btn color="blue darken-1" flat @click="dialog=true;typechange({name:label,type:fullSchema.type,show_as:selectedShowAsItem, default: defaultValueforType})">Save</v-btn>
              </v-card-actions> 
            </v-menu>
          </template>
        </v-textarea>
      </div>

      <!-- Simple boolean field -->
      <v-checkbox v-else-if="fullSchema.show_as === 'boolean'"
                  v-model.lazy="modelWrapper[modelKey]"
                  :label="label"
                  :name="fullKey"
                  :disabled="disabled"
                  :readonly="readonly"
                  :required="required"
                  :rules="rules"
                  @change="change"
                  @input="input"
      >
        <tooltip slot="append" :html-description="htmlDescription" />
        <template v-slot:prepend v-if="fullSchema.optional !=null && fullSchema.optional === true && !readonly" >
          <v-switch  v-model="optionalSwitch" @change="switchChanged()" style="margin-top: 0px; !important;padding-top: 0px !important;" color="green"/>
        </template>
      </v-checkbox>

      <!-- Simple strings array -->
      <v-combobox
        v-else-if="fullSchema.show_as === 'array' && fullSchema.items.show_as === 'string'"
        style="width:90%"
        v-model="modelWrapper[modelKey]"
        :name="fullKey"
        :label="label"
        :required="required"
        :rules="rules"
        :disabled="disabled"
        chips
        multiple
        append-icon=""
        @change="change"
        @input="input"
      >
        <tooltip slot="append-outer" :html-description="htmlDescription" />
        <template slot="selection" slot-scope="data">
          <v-chip close @input="modelWrapper[modelKey].splice(modelWrapper[modelKey].indexOf(data.item)); change(); input()">
            {{ data.item }}
          </v-chip>
        </template>
      </v-combobox>

      <!-- Object sub container with properties that may include a select based on a oneOf and subparts base on a allOf -->
      <div v-else-if="fullSchema.type === 'object' || fullSchema.show_as === 'choice'">
        <v-subheader v-show="fullSchema.show_as != 'choice' && modelKey !=null && modelKey!='root' && modelKey != 'currentOneOf' && fullSchema.title != '' && parentKey !='root.'" :style="foldable ? 'cursor:pointer;' :'' " class="mt-2" @click="folded = !folded">
          <v-input>
            <template v-if="fullSchema.optional !=null && fullSchema.optional === true && !readonly" v-slot:prepend>
              <v-switch  v-model="optionalSwitch" @change="switchChanged()" style="margin-top: 0px; !important;padding-top: 0px !important;" color="green"/>
            </template>
            <span :style="!readonly ? 'color:black' : '' ">{{ fullSchema.title !=null ? label:modelKey }}</span>  
          </v-input>
          <v-icon v-if="foldable && folded">
            arrow_drop_down
          </v-icon>
          <v-icon v-if="foldable && !folded">
            arrow_drop_up
          </v-icon>
        </v-subheader>
        <span v-show="parentKey =='root.' && readonly" style='text-decoration: underline;font-size: 16px;'>
          {{ fullSchema.title !=null ? label:modelKey }}
        </span>

        <v-slide-y-transition>
          <div v-show="!foldable || !folded" 
              :style="parentKey !='root.' && modelKey !='root' && isArray === undefined && !readonly ?  'border-style: groove;border-width:thin;background-color:#CFD8DC;padding:2px;padding-top:5px' :''">
            <p v-if="fullSchema.description">
              {{ fullSchema.description }}
            </p>
            <property v-for="childProp in fullSchema.properties" :key="childProp.key"
                      v-show="optionalSwitch"
                      :schema="childProp"
                      :model-wrapper="modelWrapper[modelKey]"
                      :model-root="modelRoot"
                      :model-key="childProp.key"
                      :parent-key="fullKey + '.'"
                      :required="!!(fullSchema.required && fullSchema.required.includes(childProp.key))"
                      :options="options"
                      @error="e => $emit('error', e)"
                      @change="e => $emit('change', e)"
                      @input="e => $emit('input', e)"
                      @typechange="e => $emit('typechange', e)"
                      :style="parentKey ==='root.' || modelKey==='root' ? '' : readonly ? '' : 'margin-left:15px;'"
            />

            <!-- Sub containers for allOfs -->
            <template v-if="fullSchema.allOf && fullSchema.allOf.length">
              <template v-if="!parentKey && fullSchema.allOf[0].title">
                <!-- Accordion / expansion panets at root level -->
                <!-- For vuetify 2 -->
                <v-expansion-panels
                  v-if="options.vuetifyVersion === 2"
                  :inset="options.accordionMode === 'inset'"
                  :popout="options.accordionMode === 'popout'"
                  focusable
                >
                  <v-expansion-panel v-for="(currentAllOf, i) in fullSchema.allOf" :key="i">
                    <v-expansion-panel-header style="font-weight:bold">
                      {{ currentAllOf.title }}
                    </v-expansion-panel-header>
                    <v-expansion-panel-content class="pt-2">
                      <property
                        :schema="Object.assign({}, currentAllOf, {type: 'object', title: null})"
                        :model-wrapper="subModels"
                        :model-root="modelRoot"
                        :model-key="'allOf-' + i"
                        :parent-key="parentKey"
                        :options="options"
                        @error="e => $emit('error', e)"
                        @change="e => $emit('change', e)"
                        @input="e => $emit('input', e)"
                        @typechange="e => $emit('typechange', e)"
                      />
                    </v-expansion-panel-content>
                  </v-expansion-panel>
                </v-expansion-panels>
                <!-- Vor vuetify 1 -->
                <v-expansion-panel
                  v-else
                  :inset="options.accordionMode === 'inset'"
                  :popout="options.accordionMode === 'popout'"
                  focusable
                >
                  <v-expansion-panel-content v-for="(currentAllOf, i) in fullSchema.allOf" :key="i">
                    <span slot="header" style="font-weight:bold">{{ currentAllOf.title }}</span>
                    <v-card>
                      <v-card-text>
                        <property
                          :schema="Object.assign({}, currentAllOf, {type: 'object', title: null})"
                          :model-wrapper="subModels"
                          :model-root="modelRoot"
                          :model-key="'allOf-' + i"
                          :parent-key="parentKey"
                          :options="options"
                          @error="e => $emit('error', e)"
                          @change="e => $emit('change', e)"
                          @input="e => $emit('input', e)"
                          @typechange="e => $emit('typechange', e)"
                        />
                      </v-card-text>
                    </v-card>
                  </v-expansion-panel-content>
                </v-expansion-panel>
              </template>
              <template v-else>
                <!-- simple objects if we are at first level -->
                <property
                  v-for="(currentAllOf, i) in (fullSchema.allOf || [])" :key="i"
                  :schema="Object.assign({}, currentAllOf, {type: 'object'})"
                  :model-wrapper="subModels"
                  :model-root="modelRoot"
                  :model-key="'allOf-' + i"
                  :parent-key="parentKey"
                  :options="options"
                  @error="e => $emit('error', e)"
                  @change="e => $emit('change', e)"
                  @input="e => $emit('input', e)"
                  @typechange="e => $emit('typechange', e)"
                />
              </template>
            </template>

            <!-- Sub container with a select for oneOfs -->
            <template v-if="fullSchema.show_as === 'choice'">
              <v-select
                style="width:90%"
                prepend-icon="view_list"
                v-model="currentOneOf"
                :items="fullSchema.choice"
                :disabled="disabled"
                :item-value="item => {return oneOfConstProp ? item.properties[oneOfConstProp.key].const : item.title}"
                :label="oneOfConstProp ? (oneOfConstProp.title || oneOfConstProp.key) : label"
                :required="oneOfRequired"
                :rules="oneOfRules"
                item-text="title"
                return-object
                @change="updateFormStatus();selectionChanged"
                @input="input"
                :readonly="readonly"
              >
                <tooltip slot="append-outer" :html-description="oneOfConstProp && oneOfConstProp.htmlDescription" />
              </v-select>
              <!--{{ currentOneOf }}-->
              <template v-if="currentOneOf && showCurrentOneOf">
                <property
                  :schema="Object.assign({}, currentOneOf, {title: null, type: 'object'})"
                  :model-wrapper="subModels"
                  :model-root="modelRoot"
                  :parent-key="parentKey"
                  model-key="currentOneOf"
                  :options="options"
                  @error="e => $emit('error', e)"
                  @change="e => $emit('change', e)"
                  @input="e => $emit('input', e)"
                  @typechange="e => $emit('typechange', e)"
                />
              </template>
            </template>
          </div>
        </v-slide-y-transition>
      </div>

      <!-- Tuples array sub container -->
      <div v-else-if="fullSchema.show_as === 'array' && Array.isArray(fullSchema.items)">
        <v-subheader :style="foldable ? 'cursor:pointer;' :'' " style="font-weight:1000" class="mt-2" @click="folded = !folded">
          {{ fullSchema.title !=null ? label:modelKey }}
          &nbsp;
          <v-icon v-if="foldable && folded">
            arrow_drop_down
          </v-icon>
          <v-icon v-if="foldable && !folded">
            arrow_drop_up
          </v-icon>
        </v-subheader>
        <v-slide-y-transition>
          <div v-show="!foldable || !folded">
            <p v-if="fullSchema.description">
              {{ fullSchema.description }}
            </p>
            <property v-for="(child, i) in fullSchema.items" :key="i"
                      :schema="child"
                      :model-wrapper="modelWrapper[modelKey]"
                      :model-root="modelRoot"
                      :model-key="i"
                      :parent-key="fullKey + '.'"
                      :options="options"
                      @error="e => $emit('error', e)"
                      @change="e => $emit('change', e)"
                      @input="e => $emit('input', e)"
                      @typechange="e => $emit('typechange', e)"
            />
          </div>
        </v-slide-y-transition>
      </div>

      <!-- Dynamic size array of complex types sub container -->
      <div v-else-if="fullSchema.show_as === 'array'" :key="compKey">
        <v-layout row class="mt-2 mb-1 pr-1">
          <v-subheader>{{ label }}</v-subheader>
          <v-btn small :disabled="readonly" v-if="!disabled && !(fromUrl || fullSchema.fromData)" icon color="primary" @click="updateFormStatus();modelWrapper[modelKey].push(fullSchema.items.default || defaultValue(fullSchema.items)); change(); input();compKey +=1 ;">
            <v-icon small>add</v-icon>
          </v-btn>
          <v-spacer />
          <tooltip :html-description="htmlDescription" />
        </v-layout>

        <v-container v-if="modelWrapper[modelKey] && modelWrapper[modelKey].length" grid-list-md class="pt-0 px-2">
          <v-layout row wrap>
            <draggable v-model="modelWrapper[modelKey]" handle=".handle" style="width: 100%;">
              <v-flex v-for="(itemModel, i) in modelWrapper[modelKey]" :key="i" xs12>
                <v-card class="array-card">
                  <!-- <v-card-title  class="pa-0 ma-0">
                    <v-btn :disabled="readonly" v-if="!disabled && fullSchema['x-sortable'] !== false" icon class="handle">
                      <v-icon>reorder</v-icon>
                    </v-btn>
                    <span v-if="itemTitle && modelWrapper[modelKey][i]">{{ modelWrapper[modelKey][i][itemTitle] }}</span>
                    <v-spacer />
                    
                  </v-card-title> -->
                  <v-card-text>
                    <v-btn style="position: relative; float: right; color: white; top: -10px; right: -25px;" small :disabled="readonly" v-if="!disabled && !(fromUrl || fullSchema.fromData)" icon @click="modelWrapper[modelKey].splice(i, 1); change(); input()">
                      <v-icon small color="black">delete</v-icon>
                    </v-btn>
                    <property :schema="fullSchema.items"
                              :model-wrapper="modelWrapper[modelKey]"
                              :model-root="modelRoot"
                              :model-key="i"
                              :parent-key="`${fullKey}.`"
                              :options="options"
                              @error="e => $emit('error', e)"
                              @change="e => $emit('change', e)"
                              @input="e => $emit('input', e)"
                              @typechange="e => $emit('typechange', e)"
                              :is-array="true"
                    />
                  </v-card-text>
                </v-card>
              </v-flex>
            </draggable>
          </v-layout>
        </v-container>
      </div>


      <p v-else-if="options.debug">
        Unsupported type "{{ fullSchema.show_as }}" - {{ fullSchema }}
      </p>
    </div>
  </v-form>
</template>

<script>
import { type } from 'os'
import SelectIcon from './SelectIcon.vue'
import SelectItem from './SelectItem.vue'
import Tooltip from './Tooltip.vue'
import schemaUtils from '../utils/schema'
import selectUtils from '../utils/select'
import VueIp from './IPComponent.vue'
import DateTimeComponent from './DateTimeComponent.vue'
const matchAll = require('match-all')
const md = require('markdown-it')()

export default {
  name: 'Property',
  components: { SelectIcon, SelectItem, Tooltip, VueIp, 'v-datetime-picker':DateTimeComponent },
  props: ['schema', 'modelWrapper', 'modelRoot', 'modelKey', 'parentKey', 'required', 'options', 'firsttime', 'isArray' ,'isNewform'],
  data() {
    return {
      ready: false,
      menu: false,
      rawSelectItems: null,
      selectItems: null,
      q: '',
      currentOneOf: null,
      showCurrentOneOf: true,
      fromUrlParams: {},
      loading: false,
      folded: true,
      showColorPicker: false,
      dialog: true,
      subModels: {}, // a container for objects from root oneOfs and allOfs
      // maintain vuetify1 compatibility without triggering warning on flat attribute for vuetify 2
      oldFlat: `
        background-color: none !important;
        border-color: none !important;
        `,
      selectedShowAsItem:'',
      showAsItems:["timestamp","coordinates","ip"],
      showAsItemsforString: ["hex","octet string"],
      optionalSwitch: true,
      tempObject:{},
      compKey: 0,
      defaultValueforType: ''
    }
  },
  filters:{

    timestampfilter(val){

      var d = new Date(0);
      d.setUTCSeconds(new Date(val).getTime());

      return d;

    },
    convertHextoAscii(val) {
      var hex = val.toString();
      var asciiString = "";
      for (var i = 0; i < hex.length; i += 2) {
        asciiString += String.fromCharCode(parseInt(hex.substr(i, 2), 16));
      }
      return asciiString;
    },
    convertCoordinates(val){
      return val * .000001 ;
    }
    
  },
  computed: {
    fullSchema() {
      return schemaUtils.prepareFullSchema(this.schema, this.modelWrapper, this.modelKey)
    },
    htmlDescription() {
      return (this.fullSchema && this.fullSchema.description) ? md.render(this.fullSchema.description) : null
    },
    fullKey() { return (this.parentKey + this.modelKey ).replace('root.', '') },
    label() { 	
      return typeof this.modelKey === 'string' ? this.modelKey : '' 	
      //return (typeof this.modelKey === 'string' ? this.modelKey : '') || this.fullSchema.title 	
    },
    rules() {
      return schemaUtils.getRules(this.fullSchema, this.required, this.options)
    },
    fromUrl() {
      return !!(this.fullSchema['x-fromUrl'] && this.fullSchema['x-fromUrl'].indexOf('{q}') === -1)
    },
    fromUrlWithQuery() {
      return !!(this.fullSchema['x-fromUrl'] && this.fullSchema['x-fromUrl'].indexOf('{q}') !== -1)
    },
    fromUrlKeys() {
      // Look for variable parts in the URL used to fetch data
      if (!this.fullSchema['x-fromUrl']) return null
      return matchAll(this.fullSchema['x-fromUrl'], /\{(.*?)\}/g).toArray().filter(key => key !== 'q')
    },
    itemKey() {
      return this.fullSchema['x-itemKey'] || 'key'
    },
    itemTitle() {
      return this.fullSchema['x-itemTitle'] || 'title'
    },
    itemIcon() {
      return this.fullSchema['x-itemIcon'] || (this.fullSchema['x-display'] === 'icon' ? this.itemKey : null)
    },
    disabled() {
      return this.options.disableAll
    },
    readonly() {
      return this.options.readonly
    },
    foldable() {
      return this.options.autoFoldObjects && this.parentKey
      // && this.fullSchema.title
    },
    oneOfConstProp() {
      if (!this.fullSchema.oneOf) return
      const props = this.fullSchema.oneOf[0].properties
      const key = Object.keys(props).find(p => !!props[p].const)
      if (!key) return
      return { ...props[key], key, htmlDescription: md.render(props[key].description || '') }
    },
    oneOfRequired() {
      return !!(this.oneOfConstProp && this.fullSchema && this.fullSchema.required && this.fullSchema.required.find(r => r === this.oneOfConstProp.key))
    },
    oneOfRules() {
      const rules = []
      if (this.oneOfRequired) rules.push((val) => (val !== undefined && val !== null && val !== '') || this.options.requiredMessage)
      return rules
    },
    oneOfSelect() {
      return schemaUtils.isOneOfSelect(this.fullSchema)
    }
  },
  watch: {
    q() {
      // This line prevents reloading the list just after selecting an item in an auto-complete
      if (this.modelWrapper[this.modelKey] && this.modelWrapper[this.modelKey][this.itemTitle] === this.q) return
      this.fetchSelectItems()
    },
    fullSchema: {
      handler() {
        if (this.fullSchema && JSON.stringify(this.fullSchema) !== this.lastFullSchema) {
          this.lastFullSchema = JSON.stringify(this.fullSchema)
          this.initFromSchema()
          this.cleanUpExtraProperties()
          this.applySubModels()
          this.ready = true
        }
      },
      immediate: true
    },
    currentOneOf(newVal, oldVal) {
      // use this boolean to force removing then re-creating the object property
      // base on the currentOneOf sub schema. If we don't the component is reused and reactivity creates some difficult bugs.
      this.showCurrentOneOf = false
      this.$nextTick(() => {
        this.showCurrentOneOf = true
        if (!this.currentOneOf) this.$set(this.subModels, 'currentOneOf', {})
        this.cleanUpExtraProperties()
      })
    },
    subModels: {
      handler() {
        this.cleanUpExtraProperties()
        this.applySubModels()
      },
      deep: true
    },
    rawSelectItems: {
      handler() {
        this.updateSelectItems()
      },
      immediate: true
    }
  },
  mounted(){
    this.$refs.form.validate();
  },
  methods: {
    updateFormStatus(){
      localStorage.isNewForm ="true";
    },
    check(event){
      return 100;

    },
    updateSelectItems() {
      const selectItems = selectUtils.getSelectItems(this.rawSelectItems, this.fullSchema, this.modelWrapper, this.modelKey, this.itemKey)
      if (this.fullSchema['x-display'] === 'list') {
        selectUtils.fillList(this.fullSchema, this.modelWrapper, this.modelKey, selectItems, this.itemKey)
      } else {
        selectUtils.fillSelectItems(this.fullSchema, this.modelWrapper, this.modelKey, selectItems, this.itemKey)
      }

      // we check for actual differences in order to prevent infinite loops
      if (JSON.stringify(selectItems) !== JSON.stringify(this.selectItems)) {
        this.selectItems = selectItems
      }
    },
    reset(typename){
      this.$emit('typechange', { name: typename, remove_entry: true});
    },
    typechange(payload){
      this.$emit('typechange', { name:payload.name,type:payload.type, show_as:payload.show_as, default:payload.default, remove_entry: false});
    },
    switchChanged(){
      if(this.optionalSwitch === true){
        
        if(this.fullSchema.type === 'object'){
          this.modelWrapper[this.modelKey] = JSON.parse(localStorage[this.modelKey]);
        }
        else{
          this.modelWrapper[this.modelKey]= this.tempObject[this.modelKey];
        }
        
      }
      else{
        localStorage[this.modelKey] = JSON.stringify(this.modelWrapper[this.modelKey]);
        this.tempObject[this.modelKey]= this.modelWrapper[this.modelKey];
        delete this.modelWrapper[this.modelKey];
      }
    },
    change() {
      this.updateSelectItems()
      this.$emit('change', { key: this.fullKey.replace(/allOf-([0-9]+)\./g, ''), model: this.modelWrapper[this.modelKey] })
    },
    selectionChanged(){
      this.modelWrapper[this.modelKey] ={};
    },
    convertAsciitoHex(val) {
      //for Octect string
      if(val!=null){
        var arr = [];
        for (var i = 0, l = val.length; i < l; i++) {
          var hex = Number(val.charCodeAt(i)).toString(16);
          arr.push(('0'+ hex).slice(-2));
        }
        return arr.join("");
      }
    },
    ipChange(ip, index, valid) {
      //Logic: Convert IP to Integer
      var val =
        ip.split(".").reduce(function(ipInt, octet) {
          return (ipInt << 8) + parseInt(octet, 10);
        }, 0) >>> 0;

      if (valid) {
          this.modelWrapper[this.modelKey]= val
          this.$emit('change', { key: this.fullKey.replace(/allOf-([0-9]+)\./g, ''), model: this.modelWrapper[this.modelKey] });
      }
    },
    octetStringChanged(event){
      //Handle here.
      
      this.modelWrapper[this.modelKey]= this.convertAsciitoHex(event);
      this.$emit('change', { key: this.fullKey.replace(/allOf-([0-9]+)\./g, ''), model: this.modelWrapper[this.modelKey] });
    },
    coordinatesToInteger(event){
      this.modelWrapper[this.modelKey]= Math.floor(event / .000001); //multiple by 1 million
      this.$emit('change', { key: this.fullKey.replace(/allOf-([0-9]+)\./g, ''), model: this.modelWrapper[this.modelKey] });
    },
    dateTimeChanged(){
      this.modelWrapper[this.modelKey]= Math.floor(new Date(this.modelWrapper[this.modelKey]).getTime()/1000);
      this.$emit('input', { key: this.fullKey.replace(/allOf-([0-9]+)\./g, ''), model: Math.floor(new Date(this.modelWrapper[this.modelKey]).getTime()/1000) })
    },
    input() {
      
    },
    defaultValue(schema) {
      if (schema.type === 'object' && !schema['x-fromUrl'] && !schema['x-fromData']) return {}
      if (schema.type === 'array') return []
      if (schema.type === 'integer') return schema.default;
      return null
    },
    fetchSelectItems() {
      if (!this.options.httpLib) return this.$emit('error', 'No http lib found to perform ajax request')
      let url = this.fullSchema['x-fromUrl'].replace('{q}', this.q || '')
      for (const key of this.fromUrlKeys) {
        // URL parameters are incomplete
        if (this.fromUrlParams[key] === undefined) return
        else url = url.replace(`{${key}}`, this.fromUrlParams[key])
      }
      this.loading = true
      this.options.httpLib.get(url)
        .then(res => {
          const body = res.data || res.body
          const items = this.fullSchema['x-itemsProp'] ? body[this.fullSchema['x-itemsProp']] : body
          if (!Array.isArray(items)) throw new Error(`Result of http fetch ${url} is not an array`)
          this.rawSelectItems = items
          this.loading = false
        })
        .catch(err => {
          this.$emit('error', err.message)
          this.loading = false
        })
    },
    cleanUpExtraProperties() {
      // console.log('Cleanup extra properties')
      // cleanup extra properties
      if (this.fullSchema.type === 'object' && this.fullSchema.properties && Object.keys(this.fullSchema.properties).length && this.modelWrapper[this.modelKey]) {
        Object.keys(this.modelWrapper[this.modelKey]).forEach(key => {
          if (!this.fullSchema.properties.find(p => p.key === key)) {
            // console.log(`Remove key ${this.modelKey}.${key}`)
            delete this.modelWrapper[this.modelKey][key]
          }
        })
      }
    },
    applySubModels() {
      //delete this.subModels['undefined'];
      var isSetOnce= false;
      Object.keys(this.subModels).forEach(subModel => {
        Object.keys(this.subModels[subModel]).forEach(key => {          
          if (!isSetOnce && this.modelWrapper[this.modelKey][key] !== this.subModels[subModel][key]) {

            if(this.modelWrapper[this.modelKey][key] === undefined){
              this.$set(this.modelWrapper[this.modelKey], key, this.subModels[subModel][key])
              isSetOnce=true;
            }
            else{
              this.$set(this.modelWrapper[this.modelKey], key, this.subModels[subModel][key])
              isSetOnce=true;
            }
          }
        })
      })
    },
    initFromSchema() {
      if(this.fullSchema.optional) {
        localStorage.firsttime =true;
      }
      let model = this.modelWrapper[this.modelKey]
      // Manage default values
      if (model === undefined) 
      {
        if(this.fullSchema.optional && localStorage.isNewForm == 'false'){
          this.optionalSwitch = false;
          return;
        }
        model = this.defaultValue(this.fullSchema)
        if (this.fullSchema.default !== undefined)
        {
          model = JSON.parse(JSON.stringify(this.fullSchema.default))
        }
        if(this.fullSchema.optional && this.optionalSwitch){
            if(localStorage.firsttime.toString() === 'true'){
              this.$set(this.modelWrapper, this.modelKey, model);
            }
            else{
              this.optionalSwitch =false;
              delete this.modelWrapper[this.modelKey];
            }
            // Case of select based on an enum
          if ((this.fullSchema.show_as === 'array' && this.fullSchema.items.enum) || this.fullSchema.enum) {
            this.rawSelectItems = this.fullSchema.show_as === 'array' ? this.fullSchema.items.enum : this.fullSchema.enum
          }

          if(this.fullSchema.choice){
            if(Object.keys(model)[0] != undefined){
              this.currentOneOf = JSON.parse(JSON.stringify(this.fullSchema.choice.find(item => item.title === Object.keys(model)[0])));
            }
            else{
              this.currentOneOf = JSON.parse(JSON.stringify(this.fullSchema.choice))[0];
            }
          }
          
          //Init subModel for current oneOf
          if (this.currentOneOf) {
            this.$set(this.subModels, 'currentOneOf', JSON.parse(JSON.stringify(model)))
          } else {
            this.$set(this.subModels, 'currentOneOf', {})
          }
          return;
        }
        
      }
      // const always wins
      if (this.fullSchema.const !== undefined) model = this.fullSchema.const

      // color pickers do not like null values
      if (this.fullSchema.show_as === 'string' && this.fullSchema.format === 'hexcolor') model = model || ''

      // Case of a select based on ajax query
      if (this.fromUrl) this.fetchSelectItems()
      // Case of select based on an enum
      if ((this.fullSchema.show_as === 'array' && this.fullSchema.items.enum) || this.fullSchema.enum) {
        this.rawSelectItems = this.fullSchema.show_as === 'array' ? this.fullSchema.items.enum : this.fullSchema.enum
      }
      // Case of select based on a oneof on simple types
      if (this.oneOfSelect) {
        this.rawSelectItems = (this.fullSchema.show_as === 'array' ? this.fullSchema.items : this.fullSchema).oneOf.map(item => ({ ...item, [this.itemKey]: item.const || (item.enum && item.enum[0]), [this.itemTitle]: item.title }))
      }
      // Case of an auto-complete field already defined
      if (this.fromUrlWithQuery && model && model[this.itemTitle] !== undefined) {
        this.q = model[this.itemTitle]
      }
      // Case of a select based on an array somewhere in the data
      if (this.fullSchema['x-fromData']) {
        this.$watch('modelRoot.' + this.fullSchema['x-fromData'], (val) => {
          this.rawSelectItems = val
        }, { immediate: true })
      }
      // Watch the dynamic parts of the URL used to fill the select field
      if (this.fromUrlKeys) {
        this.fromUrlKeys.forEach(key => {
          if (key.startsWith('context.')) {
            this.$watch('options.' + key, (val) => {
              this.fromUrlParams[key] = val
              this.fetchSelectItems()
            }, { immediate: true })
          } else {
            this.$watch('modelRoot.' + key, (val) => {
              this.fromUrlParams[key] = val
              this.fetchSelectItems()
            }, { immediate: true })
          }
        })
      }

      // Init subModels for allOf subschemas
      if (this.fullSchema.show_as === 'object' && this.fullSchema.allOf) {
        this.fullSchema.allOf.forEach((allOf, i) => {
          this.$set(this.subModels, 'allOf-' + i, JSON.parse(JSON.stringify(model)))
        })
      }

      // Case of a sub type selection based on a oneOf
      this.currentOneOf = null
      if (this.fullSchema.show_as === 'object' && this.fullSchema.oneOf && !this.currentOneOf && this.oneOfConstProp) {
        if (model && model[this.oneOfConstProp.key]) {
          this.currentOneOf = this.fullSchema.oneOf.find(item => item.properties[this.oneOfConstProp.key].const === model[this.oneOfConstProp.key])
        } else if (this.fullSchema.default) {
          this.currentOneOf = this.fullSchema.oneOf.find(item => item.properties[this.oneOfConstProp.key].const === this.fullSchema.default[this.oneOfConstProp.key])
        }
      }
      if(this.fullSchema.choice){
        if(Object.keys(model)[0] != undefined){
          this.currentOneOf = JSON.parse(JSON.stringify(this.fullSchema.choice.find(item => item.title === Object.keys(model)[0])));
        }
        else{
          this.currentOneOf = JSON.parse(JSON.stringify(this.fullSchema.choice))[0];
        }
      }
      
      //Init subModel for current oneOf
      if (this.currentOneOf) {
        this.$set(this.subModels, 'currentOneOf', JSON.parse(JSON.stringify(model)))
      } else {
        this.$set(this.subModels, 'currentOneOf', {})
      }

      // Cleanup arrays of empty items
      if (this.fullSchema.show_as === 'array') {
        model = model.filter(item => ![undefined, null].includes(item))
      }

      this.$set(this.modelWrapper, this.modelKey, model)
    }
  }
}

</script>

<style lang="css">
.vjsf-property .array-card .v-card__text {
  padding: 6px 16px 0 16px;
}
.vjsf-property .array-card .v-card__actions {
  padding: 0 16px 6px 16px;
}

.vjsf-property .v-input--selection-controls {
  margin-top: 0;
}

.vjsf-tooltip p:last-child {
  margin-bottom: 0;
}

.vjsf-property .color-picker-trigger {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  cursor: pointer;
  display: inline-block;
  border: 2px solid #ccc;
}

.vjsf-property .color-picker-trigger-empty {
  background: linear-gradient(to top right,transparent 0,transparent calc(50% - 2.4px),#de080a 50%,transparent calc(50% + 2.4px),transparent);
}


</style>






