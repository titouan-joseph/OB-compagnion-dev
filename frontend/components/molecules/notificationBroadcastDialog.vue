<template>
  <v-dialog v-model="toggled" :retain-focus="false" max-width="600">
    <v-card>
      <v-card-title>Envoyer un message à l'asso</v-card-title>
      <v-card-text>
        <v-text-field v-model="notification.link" label="Lien"></v-text-field>
        <v-autocomplete
          v-model="notification.team"
          label="Team"
          :items="teams"
        ></v-autocomplete>
        <v-text-field
          v-model="notification.message"
          label="Message"
        ></v-text-field>
      </v-card-text>
      <v-card-actions>
        <v-btn @click="broadcast">📣</v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script lang="ts">
import Vue from "vue";
import { RepoFactory } from "~/repositories/repoFactory";
import { safeCall } from "~/utils/api/calls";
import { BroadcastNotif } from "~/utils/models/repo";

export default Vue.extend({
  name: "NotificationBroadcastDialog",
  data() {
    return {
      notification: new BroadcastNotif(),
    };
  },
  computed: {
    type() {
      return this.$accessor.dialog.type;
    },
    open() {
      return this.$accessor.dialog.open;
    },
    teams() {
      const t = this.$accessor.config.getAllConfigTeams;
      if (t) {
        return t.value.map((e: any) => e.name);
      }
      return [];
    },
    toggled: {
      get: function (): boolean | unknown {
        if (this.type == "broadcast") {
          return this.open;
        }
        if (!this.open) {
          return false;
        }
        return false;
      },
      set(val): void {
        if (!val) {
          this.$store.dispatch("dialog/closeDialog");
        }
      },
    },
  },
  methods: {
    async broadcast(): Promise<void> {
      this.toggled = false;
      this.notification.type = "broadcast";
      this.notification.date = new Date();

      await safeCall(
        this.$store,
        RepoFactory.userRepo.broadcast(this, this.notification),
        "sent",
        "server"
      );
    },
  },
});
</script>
