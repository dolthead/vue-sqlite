<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Vue + Sqlite</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true" class="ion-padding-top">
      <ion-item v-if="editItem" lines="full">
        <ion-label>Edit Item</ion-label>
        <ion-input type="text" label="Edit item" v-model="editItem.name" fill="outline" mode="md"
          @keyup.enter="updateItem" @keyup.escape="editItem = undefined" label-placement="stacked"></ion-input>
        <ion-buttons slot="end">
          <ion-button @click="updateItem" title="Save">
            <ion-icon slot="icon-only" :icon="saveOutline"></ion-icon>
          </ion-button>
          <ion-button @click="editItem = undefined" title="Cancel">
            <ion-icon slot="icon-only" :icon="closeOutline"></ion-icon>
          </ion-button>
        </ion-buttons>
      </ion-item>
      <ion-item v-else lines="full">
        <ion-label>New Item</ion-label>
        <ion-input type="text" label="New item" v-model="inputName" fill="outline" mode="md"
          @keyup.enter="addItem" label-placement="stacked"></ion-input>
        <ion-buttons slot="end">
          <ion-button @click="addItem" color="success" title="Add">
            <ion-icon slot="icon-only" :icon="addOutline"></ion-icon>
          </ion-button>
        </ion-buttons>
      </ion-item>

      <ion-item v-for="item in items" :key="item.id" lines="full" @click="setEditItem(item)" :disabled="editItem">
        <ion-label>{{ item.name }}</ion-label>
        <ion-buttons slot="end">
          <ion-button @click.stop="setEditItem(item)" :disabled="editItem" title="Edit">
            <ion-icon slot="icon-only" :icon="createOutline"></ion-icon>
          </ion-button>
          <ion-button @click.stop="deleteItem(item.id)" color="danger" :disabled="editItem" title="Delete">
            <ion-icon slot="icon-only" :icon="trashOutline"></ion-icon>
          </ion-button>
        </ion-buttons>
      </ion-item>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { CapacitorSQLite, SQLiteConnection, SQLiteDBConnection } from '@capacitor-community/sqlite';
import { IonIcon, IonButtons, IonInput, IonButton, IonItem, IonLabel, IonContent, IonHeader, IonPage, IonTitle, IonToolbar, onIonViewDidEnter, onIonViewWillLeave, toastController } from '@ionic/vue';
import { ref } from 'vue';
import { addOutline, saveOutline, trashOutline, createOutline, closeOutline } from 'ionicons/icons';

let sqlite: SQLiteConnection;
let db: SQLiteDBConnection;

const items = ref<any>();
const inputName = ref('');
const editItem = ref<any>(undefined);

onIonViewDidEnter(async () => {
  try {
    sqlite = new SQLiteConnection(CapacitorSQLite);
    const ret = await sqlite.checkConnectionsConsistency();
    const isConn = (await sqlite.isConnection("db_vite", false)).result;
    if (ret.result && isConn) {
      db = await sqlite.retrieveConnection("db_vite", false);
    } else {
      db = await sqlite.createConnection("db_vite", false, "no-encryption", 1, false);
    }

    await db.open();
    
    const query = `CREATE TABLE IF NOT EXISTS test (
        id INTEGER PRIMARY KEY NOT NULL,
        name TEXT NOT NULL
      );`;
    const res = await db.execute(query);
    if (res.changes?.changes && res.changes.changes < 0) {
      throw new Error(`Error: execute failed`);
    }

    const query3 = `select * from test`;
    const res3 = await db.query(query3);
    items.value = res3.values;
  } catch (error) {
    console.error((error as Error).message);
  } finally {
    await db.close();
  }
});

onIonViewWillLeave(async () => {
  await sqlite.closeConnection("db_vite", false)
});

const setEditItem = (item: any) => editItem.value = { ...item };

const addItem = async () => {
  if (inputName.value.trim()) {
    try {
      const newId = Date.now();
      await db.open();
      const query2 = 'insert into test (id, name) values (?, ?)';
      await db.query(query2, [newId, inputName.value]);
      items.value.push({ id: newId, name: inputName.value });
      inputName.value = '';
      toast("Item added");
    } catch (error) {
      console.error((error as Error).message);
    } finally {
      await db.close();
    }
  }
};

const updateItem = async () => {
  if (editItem.value.name.trim()) {
    try {
      await db.open();
      const query2 = 'update test set name = ? where id = ?';
      await db.query(query2, [editItem.value.name, editItem.value.id]);
      inputName.value = '';
      items.value.find((x: any) => x.id == editItem.value.id).name = editItem.value.name;
      editItem.value = undefined;
      toast("Item updated");
    } catch (error) {
      console.error((error as Error).message);
    } finally {
      await db.close();
    }
  }
};

const deleteItem = async (id: string) => {
  try {
    await db.open();
    const query = 'delete from test where id = ?';
    await db.query(query, [id]);
    items.value = items.value.filter((x: any) => x.id != id);
    toast("Item deleted");
  } catch (error) {
    console.error((error as Error).message);
  } finally {
    await db.close();
  }
};

const toast = async (message: string) => {
  const toast = await toastController.create({
      message,
      duration: 2500,
      position: "bottom",
    });
    toast.present();
};
</script>

<style lang="scss" scoped>
  ion-input {
    margin-block: 12px;
  }
</style>
