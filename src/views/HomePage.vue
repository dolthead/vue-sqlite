<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Vue + Sqlite</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-item v-if="editItem">
        <ion-input type="text" label="New item" v-model="editItem.name" @keyup.enter="updateItem"></ion-input>
        <ion-buttons slot="end">
          <ion-button @click="editItem = undefined">Cancel</ion-button>
          <ion-button @click="updateItem">Save</ion-button>
        </ion-buttons>
      </ion-item>
      <ion-item v-else>
        <ion-input type="text" label="New item" v-model="inputName" @keyup.enter="addItem"></ion-input>
        <ion-button slot="end" @click="addItem">Add</ion-button>
      </ion-item>

      <ion-item v-for="item in items" :key="item.id">
        <ion-label>{{ item.name }}</ion-label>
        <ion-buttons slot="end">
          <ion-button @click="setEditItem(item)" color="danger">Edit</ion-button>
          <ion-button @click="deleteItem(item.id)" color="danger">Del</ion-button>
        </ion-buttons>
      </ion-item>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { CapacitorSQLite, SQLiteConnection, SQLiteDBConnection } from '@capacitor-community/sqlite';
import { IonButtons, IonInput, IonButton, IonItem, IonLabel, IonContent, IonHeader, IonPage, IonTitle, IonToolbar, onIonViewDidEnter, onIonViewWillLeave } from '@ionic/vue';
import { ref } from 'vue';

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
    
    console.log(`db: db_vite opened`);
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
  } catch (error) {
    console.error((error as Error).message);
  } finally {
    await db.close();
  }
};
</script>

<style scoped></style>
