<template>
  <q-page padding>
    <div v-if="dataReady">
      <!-- Data completed -->
      <q-table
        title="List of users"
        :columns="columns"
        :rows="rows"
        row-key="id"
        :pagination="paginations"
      >
        <template #body="props">
          <q-tr :props="props">
            <q-td key="id" :props="props"> {{ props.row.id }}</q-td>
            <q-td key="fullname" :props="props"> {{ props.row.fullname }}</q-td>
            <q-td key="email" :props="props"> {{ props.row.email }}</q-td>
            <q-td key="action">
              <q-btn
                color="primary"
                flat
                round
                icon="edit"
                @click="editRecord(props.row)"
              />
              <q-btn
                color="primary"
                flat
                round
                icon="delete"
                @click="deleteRecord(props.row)"
              />
            </q-td>
          </q-tr>
        </template>
      </q-table>
    </div>
    <div v-else>
      <!-- Loading data -->
      <q-circular-progress
        indeterminate
        rounded
        size="50px"
        color="lime"
        class="q-ma-md"
      />
    </div>
    <!-- Edit Dialog -->
    <q-dialog v-model="form_edit" persistent>
      <q-card>
        <q-card-section class="row items-center">
          <q-avatar icon="delete" color="primary" text-color="white" />
          <span class="q-ml-sm">Edit user ID: {{ input.id }}</span>
        </q-card-section>

        <q-card-section>
          <div>
            <q-input
              v-model="input.fullname"
              type="text"
              label="Your Fullname"
            />
          </div>
          <div>
            <q-input v-model="input.email" type="text" label="Your Email" />
          </div>
          <div>
            <q-img
              v-if="input.img"
              :src="input.img"
              :ratio="4 / 3"
              spinner-color="primary"
              spinner-size="82px"
            ></q-img>
            <!-- file size = 1MB -->
            <q-file
              outlined
              label="Your avatar"
              v-model="uploadFile"
              accept=".jpg, .jpeg, .png"
              max-file-size="1048576"
              @rejected="onRejected"
              @update:model-value="updateFile()"
            >
              <template v-slot:append>
                <q-icon name="attach_file" />
              </template>
            </q-file>
          </div>
        </q-card-section>

        <q-card-actions align="right">
          <q-btn
            flat
            label="Cancel"
            color="primary"
            v-close-popup
            @click="onCancelEdit()"
          />
          <q-btn
            flat
            label="Edit"
            color="primary"
            v-close-popup
            @click="onEdit()"
          />
        </q-card-actions>
      </q-card>
    </q-dialog>

    <!-- Delete Dialog -->
    <q-dialog v-model="form_delete" persistent>
      <q-card>
        <q-card-section class="row items-center">
          <q-avatar icon="delete" color="primary" text-color="white" />
          <span class="q-ml-sm">Delete user ID: {{ input.id }}</span>
        </q-card-section>

        <q-card-section>
          <span class="q-ml-sm">Fullname: {{ input.fullname }}</span>
        </q-card-section>

        <q-card-actions align="right">
          <q-btn flat label="NO" color="primary" v-close-popup />
          <q-btn
            flat
            label="YES"
            color="primary"
            v-close-popup
            @click="onDelete()"
          />
        </q-card-actions>
      </q-card>
    </q-dialog>

    <DialogComponent
      v-model="showDialog"
      :propDialog="dialog"
    ></DialogComponent>
  </q-page>
</template>

<script>
import { defineComponent } from "vue";
import { useLoginUserStore } from "../stores/loginUserStrore";
import { Notify } from "quasar";
import DialogComponent from "src/components/DialogComponent.vue";

export default defineComponent({
  name: "ListUserPage",
  data() {
    return {
      dataReady: false,
      rows: [],
      columns: [
        {
          name: "id",
          label: "ID",
          field: "id",
          align: "center",
          sortable: true,
        },
        {
          name: "fullname",
          label: "FullName",
          field: "fullname",
          align: "left",
          sortable: true,
        },
        {
          name: "email",
          label: "Email",
          field: "email",
          align: "left",
          sortable: true,
        },
        { name: "action", label: "Actions", field: "action", align: "center" },
      ],
      paginations: { rowsPerPage: 7 },
      storeLogUser: useLoginUserStore(),
      input: {},
      form_edit: false,
      form_delete: false,
      showDialog: false,
      dialog: {
        icon: "",
        msg: "",
        btnType: "",
        iconColor: "primary",
      },
      uploadFile: null,
    };
  },
  methods: {
    getAllUsers() {
      const headers = { "x-access-token": this.storeLogUser.accessToken };

      this.$api
        .get("/auth", { headers })
        .then((res) => {
          if (res.status === 200) {
            this.rows = res.data.map((item) => ({
              ...item,
              img: item.img ? `${this.$RESTAPI}/file/${item.img}` : null,
            }));
          }
        })
        .catch((err) => {
          console.log(err);
          Notify.create({
            type: "negative",
            message: "Unauthorized",
          });
          this.storeLogUser.clearStorage();
          this.$router.push("/");
        });
    },
    editRecord(record) {
      this.input = record;
      this.form_edit = true;
    },
    deleteRecord(record) {
      this.input = record;
      this.form_delete = true;
    },
    onDelete() {
      const headers = { "x-access-token": this.storeLogUser.accessToken };

      this.$api
        .delete(`/auth/${this.input.id}`, { headers })
        .then((res) => {
          if (res.status === 200) {
            Notify.create({
              type: "positive",
              message: `Deleted user ID: ${res.data.id}`,
            });

            if (this.storeLogUser.userid === res.data.id) {
              this.storeLogUser.clearStorage();
              this.$router.push("/");
            } else {
              this.getAllUsers();
            }
          }
        })
        .catch((err) => {
          console.log(err);
          this.showErrDialog(err);
        });
    },
    showErrDialog(err) {
      this.showDialog = true;
      this.dialog.icon = "error";
      this.dialog.iconColor = "negative";
      this.dialog.msg = err;
      this.dialog.btnType = "Error";
    },
    onRejected(rejectedEntries) {
      const msg =
        rejectedEntries[0].failedPropValidation === "accept"
          ? "Only images (jpg, jpeg, png) are allowed"
          : "File size cannot be larger than 1MB";

      Notify.create({
        type: "negative",
        message: msg,
      });
    },
    updateFile() {
      this.input.img = this.uploadFile
        ? URL.createObjectURL(this.uploadFile)
        : null;
    },
    onCancelEdit() {
      this.getAllUsers();
    },
    async onEdit() {
      try {
        if (this.uploadFile) {
          const response = await this.uploadImage();

          if (response.status === 200) {
            await this.submitEditData(response.data.uploadFileName);
          }
        } else {
          await this.submitEditData();
        }

        this.getAllUsers();
      } catch (err) {
        console.log(err);
        this.showErrDialog(err);
      }
    },
    async uploadImage() {
      const headers = { "Content-Type": "multipart/form-data" };
      const formData = new FormData();
      formData.append("singlefile", this.uploadFile);

      return this.$api.post("/file/upload", formData, { headers });
    },
    async submitEditData(filename) {
      const img =
        filename || (this.input.img ? this.getFileName(this.input.img) : null);
      const data = {
        fullname: this.input.fullname,
        email: this.input.email,
        img,
      };
      const headers = { "x-access-token": this.storeLogUser.accessToken };

      const res = await this.$api.put(`/auth/${this.input.id}`, data, {
        headers,
      });

      if (res.status === 200) {
        Notify.create({
          type: "positive",
          message: `Updated user ID: ${res.data.id}`,
        });

        if (this.storeLogUser.userid === res.data.id) {
          this.storeLogUser.fullname = res.data.fullname;

          if (res.data.img && !this.uploadFile) {
            this.storeLogUser.avatar = `${this.$RESTAPI}/file/${res.data.img}`;
          } else {
            this.storeLogUser.avatar = "default-avatar.png";
          }
        }
      }
    },
    getFileName() {
      return this.input.img.substr(this.input.img.lastIndexOf("/") + 1);
    },
  },
  async mounted() {
    await this.getAllUsers();
    this.dataReady = true;
  },
  components: { DialogComponent },
});
</script>
