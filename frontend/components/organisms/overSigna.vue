<template>
  <div>
    <v-card :style="isDisabled ? `border-left: 5px solid green` : ``">
      <v-card-title>Signa</v-card-title>
      <v-card-subtitle
        >Contacter la signa à signaletique@24heures.org pour ajouter des lieux
        non existant dans la liste déroulante
      </v-card-subtitle>
      <v-card-text>
        <v-autocomplete
          label="Lieux"
          multiple
          :value="currentLocations"
          :items="locations"
          :disabled="isDisabled"
          @change="selectLocations"
        ></v-autocomplete>
        <v-switch v-model="isSignaRequired" label="Besoin signa"></v-switch>
        <div v-if="isSignaRequired">
          <v-data-table :headers="headers" :items="signalisation">
            <template #[`item.action`]="{ index }">
              <v-btn
                v-if="!isDisabled"
                icon
                @click="deleteSignalisation(index)"
              >
                <v-icon>mdi-delete</v-icon>
              </v-btn>
            </template>
            <template #[`item.number`]="{ index, item }">
              <v-text-field
                :value="item.number"
                label="Nombre"
                type="number"
                min="0"
                step="1"
                :disabled="isDisabled"
                @change="onItemChange($event, index)"
              ></v-text-field>
            </template>
          </v-data-table>
        </div>
      </v-card-text>
      <v-card-actions v-if="isSignaRequired && !isDisabled">
        <v-spacer></v-spacer>
        <v-btn text @click="isSignaFormOpen = true"
          >Ajouter une signalisation
        </v-btn>
      </v-card-actions>
    </v-card>

    <v-dialog v-model="isSignaFormOpen" max-width="600">
      <v-img src="img/signa.png"></v-img>
      <v-card>
        <v-card-title>Ajouter une signalisation</v-card-title>
        <v-card-text>
          <OverForm :fields="fields" @form-change="onFormChange"></OverForm>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="blue darken-1" text @click="isSignaFormOpen = false"
            >Annuler</v-btn
          >
          <v-btn color="blue darken-1" text @click="onFormSubmit"
            >Valider</v-btn
          >
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script lang="ts">
import OverForm from "@/components/overForm.vue";
import Vue from "vue";
import { Header } from "~/utils/models/Data";

export interface Data {
  isSignaRequired: boolean;
  isSignaFormOpen: boolean;
  headers: Header[];
  fields: string[];
  newSignalisation: any;
}

export default Vue.extend({
  name: "OverSigna",
  components: { OverForm },
  props: {
    isDisabled: {
      type: Boolean,
      default: () => false,
    },
  },
  data(): Data {
    return {
      isSignaRequired: false,
      isSignaFormOpen: false,

      headers: [
        { text: "nombre", value: "number" },
        { text: "type", value: "type" },
        { text: "texte à afficher", value: "text" },
        { text: "commentaire", value: "comment" },
        { text: "", value: "action" },
      ],

      fields: [],
      newSignalisation: {},
    };
  },
  computed: {
    signalisation(): any[] {
      return this.$accessor.FA.mFA.signalisation;
    },
    currentLocations(): string[] {
      if (
        this.$accessor.FA.mFA.general &&
        this.$accessor.FA.mFA.general.locations
      ) {
        return this.$accessor.FA.mFA.general.locations;
      }
      return [];
    },
    locations(): any[] {
      return this.$accessor.location.signa.map((l) => l.name);
    },
  },
  watch: {
    signalisation: {
      handler() {
        if (this.signalisation.length > 0) {
          this.isSignaRequired = true;
        }
      },
      deep: true,
    },
  },
  async mounted() {
    this.fields =
      this.$accessor.config.getConfig("fa_signalisation_form") || [];

    // get locations
    await this.$accessor.location.getAllLocations();
  },
  methods: {
    selectLocations(locations: string[]) {
      this.$accessor.FA.setLocations(locations);
    },
    onFormChange(form: any) {
      this.newSignalisation = form;
    },
    onFormSubmit() {
      this.$accessor.FA.addSignalisation(this.newSignalisation);
      this.isSignaFormOpen = false;
    },
    deleteSignalisation(index: number) {
      this.$accessor.FA.deleteSignalisation(index);
    },
    onItemChange(number: number, index: number) {
      this.$accessor.FA.updateSignalisationNumber({ index, number });
    },
  },
});
</script>

<style scoped></style>
