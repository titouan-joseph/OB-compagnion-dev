<template>
  <v-dialog v-model="mToggle" width="100%">
    <v-card>
      <v-row>
        <v-col md="5"
          ><v-img
            v-if="mUser.pp"
            :src="getPPUrl() + 'api/user/pp/' + mUser.pp"
            max-height="200px"
          ></v-img>
          <v-card-title
            >{{
              mUser.nickname
                ? mUser.nickname +
                  " ( " +
                  mUser.firstname +
                  " " +
                  mUser.lastname +
                  " )"
                : mUser.firstname + " " + mUser.lastname
            }}
          </v-card-title>
          <v-card-text>
            <OverChips :roles="mUser.team"></OverChips>
            <div v-if="hasEditingRole">
              <v-select
                v-model="newRole"
                label="ajouter un role"
                :items="teams"
              >
              </v-select>
              <v-row>
                <v-col md="3"
                  ><v-btn text @click="addRole()">Ajouter</v-btn></v-col
                >
                <v-col md="6"
                  ><v-btn text @click="deleteAllTeams()"
                    >Révoquer tous les rôles</v-btn
                  ></v-col
                >
                <v-col md="2"
                  ><v-btn text @click="saveUser()">Sauvegarder</v-btn></v-col
                >
              </v-row>
            </div>

            <v-container>
              <v-row>
                <v-col
                  md="6"
                  style="
                    display: flex;
                    justify-content: center;
                    align-items: center;
                  "
                >
                  <v-btn icon :href="'mailto:' + mUser.email">
                    <v-icon>mdi-send</v-icon>
                  </v-btn>
                  <h3>{{ mUser.email }}</h3>
                </v-col>
                <v-col md="6" style="display: flex; align-items: baseline">
                  <v-btn icon :href="'tel:+33:' + mUser.phone">
                    <v-icon>mdi-phone</v-icon>
                  </v-btn>
                  <h3>+33{{ mUser.phone }}</h3>
                </v-col>
                <v-col md="6">
                  <v-text-field
                    v-model="mUser.lastname"
                    label="Nom"
                    :disabled="!(hasEditingRole || isMe())"
                  ></v-text-field>
                </v-col>
                <v-col md="6">
                  <v-text-field
                    v-model="mUser.firstname"
                    label="Prénom"
                    :disabled="!(hasEditingRole || isMe())"
                  ></v-text-field>
                </v-col>
                <v-col md="12">
                  <v-textarea
                    v-model="mUser.comment"
                    label="Commentaire"
                    :disabled="!(hasEditingRole || isMe())"
                  ></v-textarea>
                </v-col>
                <v-col md="4">
                  <v-text-field
                    v-model="mUser.nickname"
                    label="Surnom"
                    :disabled="!(hasEditingRole || isMe())"
                  ></v-text-field>
                </v-col>
                <v-col md="4">
                  <v-text-field
                    v-model="mUser.birthdate"
                    label="Date de naissance"
                    placeholder="AAAA-MM-JJ"
                    :disabled="true"
                  ></v-text-field>
                </v-col>
                <v-col md="4" style="display: flex; align-items: baseline">
                  <p>+33&nbsp;</p>
                  <v-text-field
                    v-model="mUser.phone"
                    label="Numéro de téléphone "
                    :disabled="!(hasEditingRole || isMe())"
                    type="number"
                  ></v-text-field>
                </v-col>
                <v-col md="6">
                  <v-text-field
                    v-model="mUser.year"
                    label="Année"
                    :disabled="!hasEditingRole"
                  ></v-text-field>
                </v-col>
                <v-col md="6">
                  <v-text-field
                    v-model="mUser.departement"
                    label="Département"
                    :disabled="!hasEditingRole"
                  ></v-text-field>
                </v-col>
              </v-row>
              <v-row v-if="hasUserRole('hard')">
                <v-col md="3">
                  <v-switch
                    v-model="mUser.hasDriverLicense"
                    label="Permis"
                    :disabled="!(hasEditingRole || isMe())"
                  ></v-switch>
                </v-col>
                <v-col md="6">
                  <v-text-field
                    v-model="mUser.driverLicenseDate"
                    label="date d'obtention du permis"
                    placeholder="AAAA-MM-JJ"
                    :disabled="!hasEditingRole"
                  ></v-text-field>
                </v-col>
                <v-col md="4">
                  <v-text-field
                    v-model="mUser.balance"
                    label="Solde compte perso"
                    :disabled="true"
                  ></v-text-field>
                </v-col>
                <v-col md="4">
                  <v-switch
                    v-model="mUser.hasPayedContribution"
                    label="Cotisation"
                    :disabled="!hasEditingRole"
                  ></v-switch> </v-col
              ></v-row>
            </v-container>
            <v-container v-if="me.team.includes('humain') && mUser.friends">
              <h3>Amis :</h3>
              <v-chip
                v-for="(friend, index) in mUser.friends"
                :key="index"
                class="p-2"
                >{{ friend.username }}</v-chip
              >
              <v-card-actions class="d-flex align-start">
                <v-autocomplete
                  v-model="newFriend"
                  label="prénom.nom"
                  :items="usernames"
                  class="mx-2"
                ></v-autocomplete>
                <v-btn text @click="addFriend">Ajouter</v-btn>
              </v-card-actions>
            </v-container>
          </v-card-text>
        </v-col>
        <v-col md="7">
          <v-sheet
            class="charismaContainer"
            color="warning"
            outlined
            rounded
            width="30%"
            align="center"
          >
            <h2 class="userCharisma">
              Charisme total :
              {{ mUser.charisma === undefined ? 0 : mUser.charisma }}
            </h2></v-sheet
          >
          <AvailabilitiesCalendar :m-user="mUser" class="myCal" />
          <v-row>
            <v-col md="3">
              <v-btn text @click="saveUser()">sauvegarder</v-btn>
            </v-col>
            <v-col md="3">
              <v-btn
                v-if="hasEditingRole"
                text
                color="red"
                @click="deleteUser()"
                >supprimer</v-btn
              >
            </v-col>
            <v-col md="3">
              <v-btn
                v-if="hasEditingRole && !isValidated()"
                color="#48C52D"
                @click="validateUser()"
                >Valider (soft)</v-btn
              >
              <v-btn
                v-if="hasEditingRole && isValidated() && isSoft()"
                color="red"
                @click="$refs.confirmUnassign.open()"
                >Dévalider (soft)</v-btn>
                <ConfirmDialog ref="confirmUnassign" @confirm="unvalidateUser">
                  Ce soft sera desaffecter de <b>toutes</b> ses taches actuellement affectées !
                </ConfirmDialog>
              </v-col>
            <v-col md="3">
              <v-btn
                v-if="hasEditingRole"
                color="#65B3F3"
                @click="isEditingAvailability = true"
                >Modifier dispos</v-btn
              ></v-col
            >
          </v-row>
        </v-col>
      </v-row>
    </v-card>
    <v-dialog v-model="isEditingAvailability" width="80%"
      ><ModificationCard :user="user"
    /></v-dialog>
  </v-dialog>
</template>

<script>
import OverChips from "~/components/atoms/overChips";
import { safeCall } from "../../utils/api/calls";
import userRepo from "~/repositories/userRepo";
import { isValidated } from "~/utils/roles/index.ts";
import AvailabilitiesCalendar from "~/components/molecules/AvailabilitiesCalendar.vue";
import ModificationCard from "~/components/organisms/ModificationCard.vue";
import { RepoFactory } from "~/repositories/repoFactory";
import ConfirmDialog from "~/components/atoms/ConfirmDialog.vue";

export default {
  name: "UserInformation",
  components: { OverChips, AvailabilitiesCalendar, ModificationCard, ConfirmDialog },
  props: {
    user: {
      type: Object,
      default: () => undefined,
    },
    toggle: {
      type: Boolean,
      default: () => false,
    },
  },

  data: () => {
    return {
      newRole: undefined,
      teams: [],
      hasEditingRole: false,
      isEditingAvailability: false,
      usernames: undefined,
      newFriend: undefined,
    };
  },

  computed: {
    me() {
      return this.$accessor.user.me;
    },
    mUser: {
      get: function () {
        return this.user;
      },
      set: function (newUser) {
        this.$emit("update-user", newUser);
      },
    },
    mToggle: {
      get: function () {
        return this.toggle;
      },
      set: function (t) {
        this.$emit("update-toggle", t);
      },
    },
  },

  async mounted() {
    this.teams = this.$accessor.config.data.data
      .find((e) => e.key === "teams")
      .value.map((e) => e.name);
    this.hasEditingRole = await this.hasRole(["admin", "humain"]);
    const res = await safeCall(
      this.$store,
      RepoFactory.userRepo.getAllUsers(this)
    );
    if (res) {
      this.usernames = res.data
        .map((user) => {
          if (!user.team.includes("hard")) {
            const username = user.firstname + " " + user.lastname;
            return { text: username, value: user };
          }
        })
        .filter((item) => item);
    }
  },

  methods: {
    getPPUrl() {
      return process.env.NODE_ENV === "development"
        ? "http://localhost:2424/"
        : "";
    },
    async hasRole(roles) {
      return this.$accessor.user.hasRole(roles);
    },
    async addRole() {
      let user = this.mUser;
      if (user.team === undefined) {
        user.team = [];
      }
      if (user.team.find((role) => role === this.newRole)) {
        // already has role
      } else {
        user.team.push(this.newRole);
        this.$set(user, "team", user.team); // update rendering
        await this.$axios.put(`/user/${user._id}`, { team: user.team });
      }
    },
    async saveUser() {
      await safeCall(
        this.$store,
        userRepo.updateUser(this, this.mUser._id, this.mUser),
        "🥳",
        "une erreur est survenue lors de la sauvegarde de l'utilisateur"
      );
      this.mToggle = false;
    },
    async deleteUser() {
      this.mUser.isValid = false;
      await this.saveUser();
    },
    async deleteAllTeams() {
      this.mUser.team = ["toValidate"];
      await this.$axios.put(`/user/${this.mUser._id}`, this.mUser);
    },
    isMe() {
      return this.$accessor.user.me._id === this.mUser._id;
    },
    isValidated() {
      return isValidated(this.mUser);
    },
    isSoft() {
      return this.mUser.team.includes("soft");
    },
    hasUserRole(roles) {
      if (this.mUser.team === undefined) {
        return false;
      } else {
        return this.mUser.team.includes(roles);
      }
    },
    async validateUser() {
      if (this.mUser.team.includes("toValidate")) {
        for (var i = 0; i < this.mUser.team.length; i++) {
          if (this.mUser.team[i] === "toValidate") {
            this.mUser.team.splice(i, 1);
          }
        }
        this.mUser.team.push("soft");
        await this.$axios.put(`/user/${this.mUser._id}`, {
          team: this.mUser.team,
        });
        this.saveUser();
      }
    },
    async unvalidateUser() {
      if (this.mUser.team.includes("soft")) {
        this.mUser.team = ["toValidate"];
        this.mUser.availabilities = [];
        await this.$axios.get('/timespan/user/unassignall/' + this.mUser._id);
      }
    },
    async addFriend() {
      if (this.newFriend && this.newFriend._id) {
        //TODO: RepoFactory + safeCall
        await this.$axios
          .post(`/user/friends`, {
            from: this.mUser._id,
            to: {
              id: this.newFriend._id,
              username:
                this.newFriend.firstname + " " + this.newFriend.lastname,
            },
          })
          .then(() => {
            alert("La relation a été ajoutée !");
          });
      }
    },
  },
};
</script>

<style scoped lang="scss">
.myCal {
  height: 65vh;
  width: 50vw;
}
.charismaContainer {
  margin-top: 4vh;
  .userCharisma {
    color: rgb(0, 0, 0);
  }
}
</style>
