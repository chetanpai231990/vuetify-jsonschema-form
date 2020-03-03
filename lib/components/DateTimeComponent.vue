<template>
  <div>
    <!-- <v-subheader v-if="readonly" 
            style="height:25px !important;color:white"> 
            {{ label }} : {{ formattedDatetime }}
    </v-subheader> -->
    <v-dialog
      v-model="display"
      lazy
      full-width
      :width="width"
      :disabled="readonly"
      style="display: inline-block;"
      >
      <v-text-field
        slot="activator"
        style="width:85%"
        :label="label"
        :value="formattedDatetime"
        :disabled="disabled"
        :loading="loading"
        :error-messages="errorMessages"
        :error-count="errorCount"
        :error="error"
        :hide-details="hideDetails"
        :append-icon="appendIcon"
        :prepend-icon="prependIcon"
        readonly
      >
      
      </v-text-field>
      <v-card>
        <v-card-text style="padding: 0;">
          <v-tabs fixed-tabs v-model="activeTab">
            <v-tab key="calendar">
              <slot name="dateIcon">
                <v-icon>event</v-icon>
              </slot>
            </v-tab>
            <v-tab key="timer" :disabled="!dateSelected">
              <slot name="timeIcon">
                <v-icon>access_time</v-icon>
              </slot>
            </v-tab>
            <v-tab-item key="calendar">
              <v-date-picker
                full-width
                v-model="datePart"
                scrollable
                :locale="locale"
                actions
              >
              </v-date-picker>
            </v-tab-item>
            <v-tab-item key="timer">
              <v-time-picker
                ref="timer"
                full-width
                class="v-time-picker-custom"
                v-model="timePart"
                scrollable
                :format="timePickerFormat"
                actions
              >
              </v-time-picker>
            </v-tab-item>
          </v-tabs>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <slot name="actions" :parent="this">
            <v-btn color="grey lighten-1" flat @click.native="clearHandler">{{
              clearText
            }}</v-btn>
            <v-btn color="green darken-1" flat @click="okHandler">{{
              okText
            }}</v-btn>
          </slot>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <v-menu v-if="!readonly" transition="slide-x-transition" bottom left :close-on-content-click.stop='dialog'>
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
              <v-text-field  label="Current Type" readonly value="timestamp"></v-text-field>
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
        <v-btn color="blue darken-1" flat @click="dialog=true;typechange({name:label,type:'timestamp',show_as:selectedShowAsItem, default: defaultValueforType})">Save</v-btn>
      </v-card-actions> 
    </v-menu>
    
  </div>
  
</template>

<script>
import moment from "moment";

const DEFAULT_DATE_FORMAT = "YYYY-MM-DD";
const DEFAULT_TIME_FORMAT = "HH:mm";
const DEFAULT_TIME = "00:00";

export default {
  name: "v-datetime-picker",
  model: {
    prop: "datetime",
    event: "input"
  },
  props: {
    datetime: {
      type: [Date, String],
      default: null
    },
    label: {
      type: String,
      default: ""
    },
    width: {
      type: Number,
      default: 320
    },
    format: {
      type: String,
      default: "YYYY-MM-DD HH:mm:ss"
    },
    timePickerFormat: {
      type: String,
      default: "24hr"
    },
    locale: {
      type: String,
      default: "en-us"
    },
    clearText: {
      type: String,
      default: "CANCEL"
    },
    okText: {
      type: String,
      default: "OK"
    },
    disabled: {
      type: Boolean,
      default: false
    },
    loading: {
      type: Boolean,
      default: false
    },
    errorMessages: {
      type: [String, Array],
      default: () => []
    },
    errorCount: {
      type: [Number, String],
      default: 1
    },
    error: {
      type: Boolean,
      default: false
    },
    hideDetails: {
      type: Boolean,
      default: false
    },
    appendIcon: {
      type: String
    },
    prependIcon: {
      type: String
    },
    readonly: {
      type: Boolean
    }
  },
  data() {
    return {
      display: false,
      dateSelected: false,
      timeSelected: false,
      activeTab: 0,
      selectedDatetime: null,
      selectedShowAsItem:'',
      showAsItems:["timestamp","coordinates","ip","integer"],
      dialog: true,
      defaultValueforType: ''
    };
  },
  mounted() {

    if (this.datetime instanceof Date) {
      this.selectedDatetime = this.datetime;
    } else if (
      typeof this.datetime === "string" ||
      this.datetime instanceof String
    ) {
      this.selectedDatetime = moment(this.datetime, this.format);
    }

    this.$emit("input", this.selectedDatetime);
  },
  computed: {
    datePart: {
      get() {
        let val = this.selectedDatetime
          ? moment(this.selectedDatetime).format(DEFAULT_DATE_FORMAT)
          : "";
        return val;
      },
      set(val) {
        this.dateSelected = true;
        this.activeTab = 1;

        let date = moment(val, DEFAULT_DATE_FORMAT);
        let hour = this.selectedDatetime
          ? moment(this.selectedDatetime).hour()
          : 0;
        let minute = this.selectedDatetime
          ? moment(this.selectedDatetime).minute()
          : 0;
        let input = moment()
          .year(date.year())
          .month(date.month())
          .date(date.date())
          .hour(hour)
          .minute(minute)
          .second(0);
        this.selectedDatetime = input.toDate();
      }
    },
    timePart: {
      get() {
        let val = this.selectedDatetime
          ? moment(this.selectedDatetime).format(DEFAULT_TIME_FORMAT)
          : DEFAULT_TIME;
        return val;
      },
      set(val) {
        this.timeSelected = true;

        let time = moment(val, DEFAULT_TIME_FORMAT);
        let input = moment(this.selectedDatetime)
          .hour(time.hour())
          .minute(time.minute())
          .second(0);
        this.selectedDatetime = input.toDate();
      }
    },
    formattedDatetime() {

      var d = new Date(0);
      d.setUTCSeconds(
        Number(this.datetime) === 1 || Number(this.datetime) === 0
          ? Math.floor(new Date().getTime()/1000)
          : Math.floor(new Date(this.datetime).getTime())
      );
      this.datetime = d;
      return this.datetime ? moment(this.datetime).format(this.format) : "";
    }
  },
  methods: {
    typechange(payload){
      this.$emit('showAsChanged', { name:payload.name,type:payload.type, show_as:payload.show_as, default:payload.default, remove_entry: false});
    },
    okHandler() {
      this.display = false;
      this.activeTab = 0;
      this.$refs.timer.selectingHour = true;
      this.$emit("input", this.selectedDatetime);
    },
    clearHandler() {
      this.display = false;
      this.activeTab = 0;
      this.$refs.timer.selectingHour = true;

      //this.$emit("input", null);
    }
  }
};
</script>

