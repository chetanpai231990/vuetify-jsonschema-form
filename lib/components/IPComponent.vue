<template>
  <span
    class="vue-ip"
    :class="{'material-theme': theme === 'material',
      active: active,
      valid: valid
    }"
  >
    <div style="color:grey">
      <slot></slot>
    </div>
    <div class="segment" style="display: inline-block" v-for="(segment, index) in ipCopy" :key="index">
      <input
        style="text-align: center;  width: 40px;  outline: none;  border: none;  color: black;"
        type="number"
        v-model="ipCopy[index]"
        :disabled="readonly"
        :placeholder="placeholderPos(index)"
        maxlength="3"
        @paste="paste($event)"
        @keydown="ipKeydown($event, index)"
        @focus="ipFocus(index)"
        @blur="blur"
        @change="ipChanged(index)"
        ref="ipSegment"
      />
    </div>
  </span>
</template>
<script>
export default {
  props: {
    onChange: Function,
    ip: {
      required: true,
      type: String
    },
    placeholder: {
      type: [Boolean],
      default: false
    },
    theme: {
      type: [String, Boolean],
      default: false
    },
    index: {
      required: true,
      type: Number
    },
    readonly: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      ipCopy: ["1", "1", "1", "1"],
      valid: false,
      active: false
    };
  },
  beforeMount() {
    // Copy the values over
    this.copyValue(this.ip);
  },
  watch: {
    /**
     * Watch the IP prop for changes and update internally
     */
    ip(newIp) {
      this.copyValue(newIp);
    }
  },
  methods: {
    /**
     * Placeholder with dummy IP
     */
    placeholderPos(segment) {
      // No placeholder
      if (!this.placeholder) return "";

      // Dummy IP placeholder
      switch (segment) {
        case 0:
          return "192";
        case 1:
          return "168";
        case 2:
          return "0";
        case 3:
          return "1";
      }
    },

    ipChanged(index){
      console.log('Ip Chnaged');
      // If its a 0 then always move to the next segment, if not work out if we need to move first
      if (this.ipCopy[index] === "0") {
        this.moveToNextIpSegment(index, false);
      } else if (this.ipCopy[index] < "0") {
        this.ipCopy[index] = "0";
        this.moveToNextIpSegment(index, false);
      }else if (
        this.ipCopy[index].length === 3 &&
        this.ipCopy[index] >= "255"
      ) {
        this.ipCopy[index] = "255";
        this.moveToNextIpSegment(index);
      } else if (this.ipCopy[index].length > 3 && index === 3) {
        this.ipCopy[index] = "255";
        this.moveToNextIpSegment(index);
      } else {
        this.moveToNextIpSegment(index);
      }

      // Update the change
      this.changed();
    },

    /**
     * On focus clear the current box
     */
    ipFocus(index) {
      this.active = true;

      // Clear it
      //this.ipCopy[index] = "";

      // Update the change
      this.changed();
    },

    /**
     * Clear both inputs
     */
    clearAll() {
      this.ipCopy = ["", "", "", ""];
      this.valid = false;
    },

    /**
     * Clicked off the IP
     */
    blur() {
      this.active = false;
    },

    /**
     * Paste in an IP (with or without a port)
     */
    paste(event) {
      // Focus on first el
      this.$refs.ipSegment[0].focus();

      // Get clipboard text
      let pasteText = event.clipboardData.getData("text/plain");

      // Check if we have a port or not
      let portPos = pasteText.indexOf(":");

      // If we have ports turned off, remove the port and only update the IP value
      if (this.port === false) {
        console.warn(
          "A IP address with a port has been entered but this module has no port attribute. Please enable it update the port."
        );

        this.clearAll();

        let ipAndPort = pasteText.split(":");
        this.copyValue(ipAndPort[0], false);

        // Blur off input
        this.$refs.ipSegment[0].blur();

        return;
      }

      // Based on if we have a port or not
      switch (portPos) {
        case -1:
          this.copyValue(pasteText, null);
          this.changed();

          // Blur off input
          this.$refs.ipSegment[0].blur();

          break;
        default:
          let ipAndPort = pasteText.split(":");
          this.copyValue(ipAndPort[0], ipAndPort[1]);
          this.changed();

          // Blur off input
          this.$refs.ipSegment[0].blur();

          break;
      }
    },

    /**
     * Port keydown event
     */
    portKeydown() {
      let keyCode = event.keyCode || event.which;

      // Return or left on keypad
      if (keyCode === 8 || keyCode === 37) {
        // If there is nothing within the selected input go back to the last IP segment
        if (
          this.portCopy &&
          this.portCopy.length === 0 &&
          this.portCopy !== undefined
        )
          this.$refs.ipSegment[3].focus();
      }

      setTimeout(this.changed);
    },

    /**
     * Run on keydown
     */
    ipKeydown(event, index) {
      let keyCode = event.keyCode || event.which;
      // Return or left on keypad
      if (keyCode === 8 || keyCode === 37) {
        // If there is nothing within the selected input go the the one before it
        if (
          this.ipCopy[index].length === 0 &&
          this.ipCopy[index - 1] !== undefined
        )
          this.$refs.ipSegment[index - 1].focus();
      }
      setTimeout(() => {
        // If its a 0 then always move to the next segment, if not work out if we need to move first
        if (this.ipCopy[index] === "0") {
          this.moveToNextIpSegment(index, false);
        } else if (this.ipCopy[index] < "0") {
          this.ipCopy[index] = "0";
        }else if (
          this.ipCopy[index].length === 3 &&
          this.ipCopy[index] >= "255"
        ) {
          this.ipCopy[index] = "255";
          this.moveToNextIpSegment(index);
        } else if (this.ipCopy[index].length > 3 && index === 3) {
          this.ipCopy[index] = "255";
          this.moveToNextIpSegment(index);
        } else {
          this.moveToNextIpSegment(index);
        }

        // Update the change
        this.changed();
      });
    },

    /**
     * Work out if we need to move to the next IP segment or not
     */
    moveToNextIpSegment(index, ifOverThree = true) {
      /**
       * If there is 3 characters check if there is another segment, if there is focus on it.
       */
      if (ifOverThree) {
        if (
          this.ipCopy[index].length >= 3 &&
          this.ipCopy[index + 1] !== undefined
        )
          this.$refs.ipSegment[index + 1].focus();
      } else if (!ifOverThree) {
        if (this.ipCopy[index + 1] !== undefined)
          this.$refs.ipSegment[index + 1].focus();
      }
    },

    /**
     * Update the controller with changed IP
     */
    changed(ip = this.ipCopy) {
      let ipLocal = this.arrayToIp(ip);
      this.onChange(ipLocal, this.index, this.validateIP(ip));
    },

    /**
     * Copy prop into local copy
     */
    copyValue(ip) {
      if (ip) this.ipToArray(ip);

      // Update if its valid locally
      this.valid = this.validateIP(this.ipCopy);

      // Report right back with if its valid or not
      this.changed();
    },

    /**
     * Convert the IP address string to an array of values
     */
    ipToArray(ip) {
      //Conver Integer to IP address
      ip =
        (ip >>> 24) +
        "." +
        ((ip >> 16) & 255) +
        "." +
        ((ip >> 8) & 255) +
        "." +
        (ip & 255);

      let segments = [];
      ip.split(".").map(segment => {
        if (isNaN(segment) || segment < 0 || segment > 255) segment = 255;
        segments.push(segment);
      });

      // If something is not valid clear it all
      if (segments.length !== 4) {
        console.error("Not valid, so clearing ip", segments);
        this.clearAll();
      } else this.ipCopy = segments;
    },

    /**
     * Convert the array of IP segments back to a string
     */
    arrayToIp(ipArray) {
      return ipArray.join(".");
    },

    /**
     * validates the IP address
     *
     * @returns Boolean
     */
    validateIP(ip) {
      let ipCheck = this.arrayToIp(ip);
      return /^(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/.test(
        ipCheck
      );
    }
  }
};
</script>
<style scoped>
.vue-ip {
  position: relative;
  display: inline-block;
  text-align: left;
}
.vue-ip.material-theme {
  transition: all 0.15s ease-in-out;
  border-bottom: 1px solid rgba(250, 250, 250, 0.6);
  padding-top: 12px;
}
.vue-ip.material-theme .label {
  display: block;
  transition: all 0.15s ease-in-out;
  position: absolute;
  width: 100%;
  font-size: 0.6rem;
  top: -20px;
  color: bl;
}
.vue-ip.material-theme .segment:after {
  color: black;
}
.vue-ip.material-theme.active.valid.material-theme {
  border-bottom-color: #409eff;
}
.vue-ip.material-theme.active.valid.material-theme .segment:after {
  color: #409eff;
}
.vue-ip.material-theme.active.valid.material-theme .label {
  color: #409eff;
}
.vue-ip.material-theme.active:not(.valid).material-theme {
  border-bottom-color: #f25d59;
}
.vue-ip.material-theme.active:not(.valid).material-theme .segment:after {
  color: #f25d59;
}
.vue-ip.material-theme.active:not(.valid).material-theme .label {
  color: #f25d59;
}
.vue-ip.material-theme input {
  background: transparent;
  font-size: inherit;
  color: #e0e0e0;
  -moz-appearance: textfield;
}
.vue-ip.material-theme input::-webkit-outer-spin-button,
.vue-ip.material-theme input::-webkit-inner-spin-button {
  -webkit-appearance: none;
}
.vue-ip.material-theme input::placeholder {
  color: rgba(224, 224, 224, 0.3);
}
.vue-ip .label {
  display: none;
}
.vue-ip.show-port .segment:last-of-type:after {
  content: "";
}

.vue-ip .segment:not(:last-of-type):after {
  content: ".";
  display: inline-block;
}

</style>

