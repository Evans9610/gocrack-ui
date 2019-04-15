<template>
  <div class="create-task">
    <h2>{{ $t('navbar.create_task') }}</h2>
    <hr>
    <!-- <pre>{{ $v }}</pre> -->
    <b-form @submit="validateBeforeSubmit" class="needs-validation" id="form-create-task">
      <TaskNameInput :v="$v.taskname" v-model="taskname"/>
      <b-form-group id="architecture" horizontal description="architecture" label="Architecture">
        <b-form-select v-model="architecture" :options="['32','64']"/>
      </b-form-group>
      <b-form-group id="selectBinary" horizontal label="Select Binary">
        <b-form-select v-model="binary" :options="binaries"/>
      </b-form-group>
      <b-form-group id="taskCase" horizontal label="Test Case">
        <b-form-input
          id="taskCase"
          type="text"
          v-model="testCase"
          novalidate
          placeholder="test case"
        />
      </b-form-group>
      <b-form-group id="useDevice" horizontal label="Use Device">
        <b-form-select v-model="useDevice" :options="['Localhost']"/>
      </b-form-group>
      <b-form-group id="taskComment" horizontal :label="$t('shared.comment')">
        <b-form-input
          id="taskComment"
          type="text"
          v-model="comment"
          novalidate
          :placeholder="$t('create_task.placeholder_comment')"
        />
      </b-form-group>
      <!-- <UserSelection v-on:usr-grant-change="updateAdditionalUsers"/> -->
      <!-- <DeviceSelection :v="$v.selectedDevices" v-model="selectedDevices"/> -->
      <!-- <AvailableEngineSelector v-model="engine"/> -->
      <!-- <TaskFileSelector :v="$v.passwordfile" v-model="passwordfile" ref="fileSelector"/> -->
      <!-- <TimeLimitInput :v="$v.timelimit" v-model="taskTimeLimit"/> -->
      <button class="btn btn-primary btn-block" type="submit">{{ $t('shared.submit') }}</button>
    </b-form>
    <!-- <UploadFileModal :isTaskFile="true" ref="fileUploadModal"/> -->
  </div>
</template>

<style>
input[type="file"] {
  color: transparent;
}
</style>

<script>
import { required, numeric, minLength } from "vuelidate/lib/validators";
import {
  isDeviceTypeSelectionValid,
  areDevicesOnSameHost
} from "@/validators/device_selection";

import Multiselect from "vue-multiselect";
import { ADD_TOAST_MESSAGE } from "@/toast";
import { mapActions } from "vuex";

import UploadFileModal from "@/components/UploadFileModal";
import DeviceSelection from "@/components/DeviceSelection";
import UserSelection from "@/components/UsersSelection";
import apitypes from "@/api/types";

import { formatNumber } from "@/filters";
import helpers from "@/helpers";

export default {
  components: {
    Multiselect,
    DeviceSelection,
    UserSelection,
    UploadFileModal
  },

  filters: {
    formatNumber
  },

  computed: {
    showHashcatOptions() {
      if (this.engine !== null && this.engine.id === 1) {
        return true;
      }
      return false;
    },

    getTaskPriorities() {
      return apitypes.TASK_PRIORITIES;
    }
  },

  mounted() {
    this.timer = setInterval(this.getBinaries, 2000);
    // this.$refs.fileSelector.$on("input", this.fileSelected);
    // this.$refs.fileUploadModal.$on("uploaded", this.fileSelected);
  },

  beforeDestroy() {
    clearInterval(this.timer);
    // this.$refs.fileSelector.$off("input", this.fileSelected);
    // this.$refs.fileUploadModal.$off("uploaded", this.fileSelected);
  },

  methods: {
    // fileSelected is called whenever a file is uploaded from the modal
    getBinaries(event) {
      fetch("http://qaq.nctu.me:1338/api/v1/files/task/")
        .then(r => r.json())
        .then(bs => {
          this.binaries = bs.map(b => b.filename);
        });
    },
    validateBeforeSubmit(event) {
      event.preventDefault();
      event.stopPropagation();

      if (false) {
        $("html, body").animate({ scrollTop: 0 }, "fast");
        this.addToast({
          text: this.$t("create_task.invalid_form"),
          type: "danger"
        });
        return;
      } else {
        this._submitform();
      }
    },

    _submitform() {
      if (this.submitting_task) {
        this.addToast({
          text: this.$t("shared.please_wait"),
          type: "warning"
        });
        return;
      }
      this.submitting_task = true;

      let enginepayload = {};
      let payload = {
        task_name: this.taskname,
        case_code: this.casecode,
        useDevice: this.useDevice,
        binary: this.binary,
        architecture: this.architecture,
        comment: this.comment,
        testCase: this.testCase
      };

      this.$gocrack
        .createTask(payload)
        .then(data => {
          console.log(data);
          this.submitting_task = false;
          // this.$router.push({
          //   name: "Task Details",
          //   params: { id: data.taskid }
          // });
        })
        .catch(error => {
          let vm = this;
          helpers.componentToastError(vm, error);
        });
      console.log("submitting task!", payload);
    },

    // attackModeChanged will clear some of the formdata fields that we no longer need when we send the data
    // off to the server
    attackModeChanged(newAttackMode) {
      switch (newAttackMode) {
        case 0:
          this.hashcat.passwordmasks = null;
          break;
        case 3:
          this.hashcat.dictionary = null;
          this.hashcat.manglingrules = null;
          break;
      }
    },

    _filter_engine_files(data, filter) {
      return data.filter(enginefile => {
        return enginefile.file_type === filter;
      });
    },

    getEngineFiles(search) {
      if (this.passwordmasks.length > 0) {
        return;
      }

      this.loading_enginefiles = true;
      this.$gocrack
        .getEngineFiles()
        .then(data => {
          this.manglingrules = this._filter_engine_files(data, "Rule(s)");
          this.passwordmasks = this._filter_engine_files(data, "Mask(s)");
          this.dictionaries = this._filter_engine_files(data, "Dictionary");
          this.loading_enginefiles = false;
        })
        .catch(error => {
          this.loading_enginefiles = false;
          let vm = this;
          helpers.componentToastError(vm, error);
        });
    },

    updateAdditionalUsers(data) {
      this.additionalUsers = data;
    },

    ...mapActions({
      addToast: ADD_TOAST_MESSAGE
    })
  },

  data() {
    return {
      hashcat: {
        attack_mode: apitypes.HASHCAT_ATTACK_MODES[0],
        // Selected file
        dictionary: null,
        manglingrules: null,
        passwordmasks: null,
        hashtype: null
      },
      loading_enginefiles: false,
      submitting_task: false,
      // List of all the files
      passwordmasks: [],
      dictionaries: [],
      manglingrules: [],

      // generic fields
      engine: apitypes.GOCRACK_ENGINES[0],
      taskname: "",
      casecode: "",
      comment: null,
      passwordfile: null,
      selectedDevices: [],
      additionalUsers: [],
      priority: 1,
      taskTimeLimit: null,
      testCase: null,
      useDevice: null,
      architecture: 32,
      binary: null,
      binaries: []
    };
  },

  validations() {
    let validations = {};

    // Switch validation based on if hashcat is the selected engine
    if (this.showHashcatOptions) {
      let hcvalid = {
        dictionary: {},
        passwordmasks: {},
        hashtype: {
          required
        }
      };
      // Again, switch validations based on the attack mode!
      switch (this.hashcat.attack_mode.id) {
        case apitypes.HASHCAT_ATTACKMODE_STRAIGHT:
          hcvalid.dictionary = {
            required
          };

          break;
        case apitypes.HASHCAT_ATTACKMODE_BF:
          hcvalid.passwordmasks = {
            required
          };
          break;
      }
      validations.hashcat = hcvalid;
    }
    return validations;
  }
};
</script>
