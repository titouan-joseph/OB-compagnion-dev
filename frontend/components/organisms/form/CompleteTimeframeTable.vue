<template>
  <div>
    <v-data-table
      dense
      :headers="headers"
      :items="timeframes"
      :hide-default-footer="true"
      :items-per-page="-1"
    >
      <template #[`item.dateStart`]="{ item }">
        {{ new Date(item.start).toDateString() }}
      </template>
      <template #[`item.dateEnd`]="{ item }">
        {{ new Date(item.end).toDateString() }}
      </template>
      <template #[`item.start`]="{ item }">
        {{ formatMilliToLocalTime(item.start) }}
      </template>
      <template #[`item.end`]="{ item }">
        {{ formatMilliToLocalTime(item.end) }}
      </template>
      <template #[`item.action`]="{ index }">
        <div v-if="!isDisabled">
          <v-btn icon>
            <v-icon @click="deleteTimeframe(index)">mdi-trash-can</v-icon>
          </v-btn>
          <v-btn
            v-if="!isDisabled"
            icon
            @click="editTimeframeRequirements(index)"
          >
            <v-icon>mdi-account-multiple-plus-outline</v-icon>
          </v-btn>
          <v-btn icon @click="editTimeframe(index)">
            <v-icon>mdi-pencil</v-icon>
          </v-btn>
        </div>
      </template>
      <template #[`item.required`]="{ index, item }">
        <v-chip-group column>
          <template v-for="(req, i) in item.required">
            <v-chip
              v-if="!isRequiredInConflict(req, item)"
              :key="req._id"
              close
              @click:close="removeRequirement(i, index)"
              @click="openUserCalendar(req)"
            >
              {{ formatText(req) }}
            </v-chip>
            <v-tooltip v-else :key="`${req._id}_conflict`" top>
              <template #activator="{ on, attrs }">
                <v-chip
                  close
                  :color="getConflictHelper(req, item).color"
                  v-bind="attrs"
                  v-on="on"
                  @click:close="removeRequirement(i, index)"
                  @click="openUserCalendar(req)"
                >
                  {{ formatText(req) }}
                </v-chip>
              </template>
              <span>{{ getConflictHelper(req, item).helperText }}</span>
            </v-tooltip>
          </template>
        </v-chip-group>
      </template>
      <!-- Partition displays "-" if it is not defined or false and the slot time else -->
      <template #[`item.toSlice`]="{ item }"
        >{{
          item.toSlice === undefined || item.toSlice === false
            ? "-"
            : `${item.sliceTime}h`
        }}
      </template>
    </v-data-table>

    <!-- Dialog to edit a single timeslot -->
    <v-dialog v-model="isEditDialogOpen" max-width="600">
      <v-card>
        <v-card-title>
          <span class="headline">Éditer une plage</span>
        </v-card-title>
        <v-form v-model="validTimeframeEdit" lazy-validation>
          <v-card-text>
            <v-text-field
              v-model="mTimeframe.start"
              label="Début"
              type="datetime-local"
              :disabled="isDisabled"
              :rules="dateTimeValidationRules()"
              required
            ></v-text-field>
            <v-text-field
              v-model="mTimeframe.end"
              label="Fin"
              type="datetime-local"
              :disabled="isDisabled"
              :rules="dateTimeValidationRules()"
              required
            ></v-text-field>
            <v-checkbox
              v-model="mTimeframe.toSlice"
              label="Découper"
            ></v-checkbox>
            <v-slider
              v-model="mTimeframe.sliceTime"
              label="Nombre d'heures par découpage"
              :disabled="!mTimeframe.toSlice"
              min="0.5"
              max="4"
              step="0.5"
              thumb-label="always"
            ></v-slider>
          </v-card-text>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="blue darken-1" text @click="isEditDialogOpen = false">
              Annuler
            </v-btn>
            <v-btn
              color="blue darken-1"
              text
              :disabled="isDisabled || !validTimeframeEdit"
              @click="updateTimeframe"
            >
              Valider
            </v-btn>
          </v-card-actions>
        </v-form>
      </v-card>
    </v-dialog>

    <!-- Orga selection pop up -->
    <v-dialog v-model="requireDialog" max-width="600">
      <v-card>
        <v-card-title>Orga requis</v-card-title>
        <v-card-text>
          <h3>Ajouter un orga</h3>
          <OverField
            :field="{
              key: 'validated_user',
              label: 'orga',
              type: 'validated_user',
            }"
            @value="updateUser"
          ></OverField>
          <v-btn text :disabled="!required.user._id" @click="addUser"
            >demander l'orga
          </v-btn>
          <OverField
            :field="{ key: 'team', label: 'team', type: 'teams' }"
            @value="updateTeam"
          ></OverField>
          <v-text-field
            v-model="required.amount"
            type="number"
            label="Nombre"
            value="1"
            :rules="orgaAmountRules"
          ></v-text-field>
          <v-btn
            text
            :disabled="!required.team || !required.amount"
            @click="addTeam"
            >demander la team
          </v-btn>
        </v-card-text>
      </v-card>
    </v-dialog>
  </div>
</template>

<script>
import OverField from "../../overField";
import {
  isTFConflict,
  isTSConflict,
  isAvailabilityConflict,
} from "@/utils/models/conflicts";
import { v4 as uuidv4 } from "uuid";

const DEFAULT_SLICE_TIME = 2;

export default {
  name: "CompleteTimeframeTable",
  components: { OverField },
  props: {
    store: {
      type: Object,
      default: () => ({}),
    },
    isDisabled: {
      type: Boolean,
      default: () => false,
    },
  },
  data: () => ({
    headers: [
      { text: "Date début", value: "dateStart" },
      { text: "Début", value: "start" },
      { text: "Date fin", value: "dateEnd" },
      {
        text: "Fin",
        value: "end",
      },
      { text: "Découpage", value: "toSlice" },
      { text: "Requis", value: "required" },
      { text: "Affecté", value: "assigned" },
      { text: "Action", value: "action" },
    ],
    requireDialog: false,
    selectedTimeframeIndex: 0,
    selectedTimeframe: {},

    // indicate if the new timeframe inputs in the form is valid
    validTimeframeEdit: false,

    isEditDialogOpen: false,

    mTimeframe: {
      start: "",
      end: "",
    },
    date: {
      start: "",
      end: "",
    },

    required: {
      _id: undefined,
      type: undefined,
      team: undefined,
      amount: 1,
      user: {
        username: undefined,
        _id: undefined,
      },
      equipment: undefined,
    },

    orgaAmountRules: [
      (v) => v == "0" || !!parseInt(v) || "Entre un nombre",
      (v) => v > 0 || "Entre un nombre positif",
    ],
  }),
  computed: {
    timeframes() {
      return this.$accessor.FT.mFT.timeframes;
    },
    conflicts: function () {
      return this.$accessor.conflict.conflicts;
    },
  },
  watch: {
    isEditDialogOpen: function (val) {
      // If switched to false
      if (!val) {
        // Reset the dialog related content
        this.mTimeframe = {};
        this.selectedTimeframeIndex = -1;
        this.validTimeframeEdit = false;
      }
    },
  },
  methods: {
    /**
     * Return conflict array of a required field
     */
    requiredConflicts(req, item) {
      // team requirement cannot have conflicts
      if (req.type == "team" || item == undefined) {
        return [];
      }
      return this.conflicts.filter(
        (c) =>
          c.user == req.user._id && (c.tf1 == item._id || c.tf2 == item._id)
      );
    },
    requiredConflictsAvailability(req, item) {
      // team requirement cannot have conflicts
      if (req.type == "team" || item == undefined) {
        return [];
      }
      return this.conflicts.filter(
        (c) =>
          c.type == "availability" &&
          c.user == req.user._id &&
          c.tf1 == item._id
      );
    },

    getConflictHelper(req, item) {
      const timeframeUserConflicts = this.conflicts.filter(
        (conflict) =>
          conflict.user == req.user._id &&
          (conflict.tf1 == item._id || conflict.tf2 == item._id)
      );
      const timeframeUserTfConflicts = timeframeUserConflicts.filter(
        (conflict) => isTFConflict(conflict)
      );
      if (timeframeUserTfConflicts.length) {
        const conflictualFT = timeframeUserTfConflicts
          .map((conflict) => conflict.otherTf.ft.count)
          .join(" ");
        return {
          color: "red",
          helperText: `En conflit avec la/les FT ${conflictualFT}`,
        };
      }
      const timeframeUserTimespanConflicts = timeframeUserConflicts.filter(
        (conflict) => isTSConflict(conflict)
      );
      if (timeframeUserTimespanConflicts.length) {
        return {
          color: "purple",
          helperText: "Deja affecte sur une autre tache",
        };
      }
      const timeframeUserAvailabilityConflict = timeframeUserConflicts.filter(
        (conflict) => isAvailabilityConflict(conflict)
      );
      if (timeframeUserAvailabilityConflict.length) {
        return {
          color: "orange",
          helperText: "Pas dispo sur ce creneau",
        };
      }

      return {
        color: "grey",
        helperText: "Conflit non identifie",
      };
    },

    openUserCalendar({ type, user }) {
      if (!user?._id || type !== "user") return;
      window.open(`/calendar/${user._id}`, "_blank");
    },

    /**
     * Return if a required is in conflict
     */
    isRequiredInConflict(req, item) {
      return this.requiredConflicts(req, item).length != 0;
    },
    isRequiredAvailability(req, item) {
      return this.requiredConflictsAvailability(req, item).length != 0;
    },
    /**
     * Return the hover text of first conflict
     * Does not recheck conflicts exists
     */
    formatConflictText(req, item) {
      const conflicts = this.requiredConflicts(req, item);
      let ftCounts = "";
      conflicts.forEach((conflict) => {
        ftCounts += `${conflict.otherTf.ft.count} `;
      });

      const text = `En conflit avec la/les FT ${ftCounts}`;
      return text;
    },
    /**
     * Format date in milliseconds to ##:## 24h formatted time
     * @param milli String date in milliseconds
     * @returns String time stringified in format hh:mm
     */
    formatMilliToLocalTime(milli) {
      return new Date(milli).toLocaleTimeString("fr-FR", {
        hour: "2-digit",
        minute: "2-digit",
      });
    },
    /**
     * @param milli Number date in milliseconds
     * @returns String date stringified in format yyyy-mm-ddThh:mm in local time
     */
    formatMilliToLocalDate(milli) {
      var tzoffset = new Date(milli).getTimezoneOffset() * 60000;
      return new Date(milli - tzoffset).toISOString().slice(0, -8); // remove offset, seconds and milliseconds
    },
    /**
     * @param date String date stringified in format yyyy-mm-ddThh:mm in local time
     * @returns Number date in milli, offset is automatically set
     */
    formatLocalDatetoMilli(date) {
      return new Date(date).getTime();
    },
    /**
     * @returns Array<Function> Array of validation rules for datetime-local inputs
     */
    dateTimeValidationRules() {
      return [
        // Pattern regexp matching
        (v) =>
          /[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}/.test(v) ||
          "Entrez une date valide",
      ];
    },
    editTimeframeRequirements(index) {
      this.selectedTimeframeIndex = index;
      this.selectedTimeframe = this.timeframes[index];
      this.requireDialog = true;
    },

    updateUser(user) {
      const valid_user = {
        username: user.value.firstname + " " + user.value.lastname,
        _id: user.value._id,
      };
      this.required.user = { ...valid_user };
    },

    updateTeam(team) {
      this.required.team = { ...team }.value;
    },

    async removeRequirement(requirementIndex, timeframeIndex) {
      if (this.isDisabled) {
        alert(
          "Vous ne pouvez pas modifier les requis d'une FT déjà validée. Tu as cru que c'etait ASSOMAKER ou koi ?"
        );
        return;
      }
      await this.$accessor.FT.deleteRequirement({
        timeframeIndex,
        requirementIndex,
      });
    },

    /**
     * Open timeFrame edit dialog and store current timeFrame
     */
    editTimeframe(index) {
      this.selectedTimeframeIndex = index;
      this.mTimeframe = { ...this.timeframes[index] };

      // Adapt format because input type datetime-local does not handle milliseconds format
      this.mTimeframe.start = this.formatMilliToLocalDate(
        this.mTimeframe.start
      );
      this.mTimeframe.end = this.formatMilliToLocalDate(this.mTimeframe.end);

      // Default value for sliceTime
      this.mTimeframe.sliceTime =
        this.mTimeframe.sliceTime || DEFAULT_SLICE_TIME;

      this.isEditDialogOpen = true;
    },
    /**
     * Update Timeframe in store with the current mTimeframe's values
     */
    updateTimeframe() {
      // Format back timeFrame to the correct milliseconds format
      this.mTimeframe.start = this.formatLocalDatetoMilli(
        this.mTimeframe.start
      );
      this.mTimeframe.end = this.formatLocalDatetoMilli(this.mTimeframe.end);

      // update the store
      this.$accessor.FT.updateTimeframe({
        index: this.selectedTimeframeIndex,
        timeframe: this.mTimeframe,
      });

      // Reset
      this.isEditDialogOpen = false;
    },

    addUser() {
      if (this.selectedTimeframe.required === undefined) {
        this.selectedTimeframe.required = [];
      }
      this.required._id = uuidv4();
      this.required.amount = +this.required.amount;
      this.required.type = "user";
      let mTimeframe = { ...this.selectedTimeframe };
      this.$accessor.FT.addRequirement({
        timeframeIndex: this.selectedTimeframeIndex,
        requirement: this.required,
      });
      // this.resetRequirement();
    },

    deleteTimeframe(timeframeIndex) {
      this.$accessor.FT.deleteTimeframe(timeframeIndex);
    },

    formatText(requirements) {
      if (!requirements.type) {
        return "";
      }
      switch (requirements.type) {
        case "user":
          return requirements.user.username;
        case "team":
          return `${requirements.amount} ${requirements.team}`;
        case "equipment":
          return requirements.equipment;
        default:
          return requirements.type;
      }
    },

    resetRequirement() {
      this.required = {
        _id: undefined,
        type: undefined,
        team: undefined,
        amount: 1,
        user: {
          username: undefined,
          _id: undefined,
        },
        equipment: undefined,
      };
    },

    addTeam() {
      if (this.selectedTimeframe.required === undefined) {
        this.selectedTimeframe.required = [];
      }
      this.required._id = uuidv4();
      this.required.amount = +this.required.amount;
      this.required.type = "team";
      this.store.addRequirement({
        timeframeIndex: this.selectedTimeframeIndex,
        requirement: this.required,
      });
      // this.resetRequirement();
    },
  },
};
</script>

<style scoped></style>
