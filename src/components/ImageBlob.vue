<template>
  <div>
    <div v-if="step === 1">
      <p>Data added:</p>
      <img id="newImage" :src="newImage" alt="Image" />
    </div>

    <div v-if="step === 2">
      <div v-for="image in images" :key="image.id">
        <img :src="image.blob" alt="Saved Image" @click="setSelectedData(image.id)" />
      </div>
    </div>

    <div v-if="step === 3">
      <p>Data updated:</p>
      <img id="updateImage" :src="newImage" alt="Image" />
    </div>

    <div v-if="step === 4">
      <p>Delete Successful: {{ deletedImage }}</p>
    </div>

    <br />
    <button @click="addData">Add Data</button>
    <button @click="readData">Read Data</button>
    <button v-if="selectedImage" @click="updateData">Update Data</button>
    <button v-if="selectedImage" @click="deleteData">Delete Data</button>
  </div>
</template>

<script setup>
import { ref } from 'vue';

const dbName = "images_blob";

const State = {
  Add: 1,
  Read: 2,
  Update: 3,
  Delete: 4
}

let step = ref(0);

const newImage = ref(null);
const imageURL = ref(null);
const images = ref([]);
const selectedImage = ref(null);

const reader = new FileReader();

const setSelectedData = (imageId) => {
  selectedImage.value = imageId;
}

const fetchImageURL = async () => {
  selectedImage.value = null;
  
  try {
    await fetch("https://random.imagecdn.app/v1/image?width=500&height=500").then((response) => {
      if (response.ok) {
        return response.text();
      } else {
        throw new Error("Failed to fetch image URL");
      }
    }).then(async (response) => {
      imageURL.value = response;

      await fetchImage();
    });
  } catch (error) {
    console.error("An error occured: " + error);
  }
}

const fetchImage = async () => {
  try {
    await fetch(imageURL.value).then((response) => {
      if (response.ok) {
        return response.blob();
      } else {
        throw new Error("Failed to fetch image");
      }
    }).then((response) => {
      newImage.value = response;
      reader.readAsDataURL(newImage.value);

      return;
    });
  } catch (error) {
    console.error("An error occured: " + error);
  }
}

const addData = async () => {
  await fetchImageURL();

  const request = indexedDB.open(dbName, 2);

  request.onerror = (event) => {
    console.error("Error opening database:", event.target.error);
  };

  request.onupgradeneeded = (event) => {
    const db = event.target.result;

    const objectStore = db.createObjectStore("images", { keyPath: "id" });
    objectStore.createIndex("blob", "blob", { unique: true });
  };

  request.onsuccess = (event) => {
    const db = event.target.result;

    const addTransaction = db.transaction("images", "readwrite");
    const imageObjectStore = addTransaction.objectStore("images");

    newImage.value = reader.result;
    const imageToAdd = { id: crypto.randomUUID(), blob: newImage.value }

    const addRequest = imageObjectStore.add(imageToAdd);


    addRequest.onsuccess = (event) => {
      console.log("Data added successfully");
    };

    addRequest.onerror = (event) => {
      console.error("Error adding data: ", event.target.error);
    };

    addTransaction.oncomplete = () => {
      console.log("Add transaction completed");
      step.value = State.Add;
      db.close();
    };
  };
};

const readData = () => {
  images.value = [];
  selectedImage.value = null;

  const request = indexedDB.open(dbName, 2);

  request.onerror = (event) => {
    console.error("Error opening database:", event.target.error);
  };

  request.onsuccess = (event) => {
    const db = event.target.result;

    const readTransaction = db.transaction("images", "readonly");
    const imageObjectStore = readTransaction.objectStore("images");

    const imageCursor = imageObjectStore.openCursor();

    imageCursor.onsuccess = (event) => {
      const cursor = event.target.result;

      if (cursor) {
        images.value.push(cursor.value);
        cursor.continue();
      } else {
        console.log("Data read successfully");
        step.value = State.Read;
        db.close();
      }
    };

    imageCursor.onerror = (event) => {
      console.error("Error reading data: ", event.target.error);
      db.close();
    };
  };
}

const updateData = () => {
  const request = indexedDB.open(dbName, 2);

  request.onerror = (event) => {
    console.error("Error opening database:", event.target.error);
  };

  request.onsuccess = (event) => {
    const db = event.target.result;

    const updateTransaction = db.transaction("images", "readwrite");
    const imageObjectStore = updateTransaction.objectStore("images");

    const imageCursor = imageObjectStore.openCursor();

    imageCursor.onsuccess = (event) => {
      const cursor = event.target.result;

      if (cursor.value.id === selectedImage.value) {
        const imageToUpdate = { id: selectedImage, blob: newImage.value }
        const updateRequest = imageObjectStore.put(imageToUpdate);
    
        updateRequest.onsuccess = (event) => {
          console.log("Data updated successfully");
        };
    
        updateRequest.onerror = (event) => {
          console.error("Error updating data: ", event.target.error);
        };
    
        updateTransaction.oncomplete = () => {
          console.log("Update transaction completed");
          db.close();
        };
      } else {
        if (cursor) {
          cursor.continue();
        } else {
          console.log("Data not found");
          db.close();
        }
      }
    }

  };
}

const deleteData = () => {
  const request = indexedDB.open(dbName, 2);

  request.onerror = (event) => {
    console.error("Error opening database:", event.target.error);
  };

  request.onsuccess = (event) => {
    const db = event.target.result;

    const deleteTransaction = db.transaction("images", "readwrite");
    const deleteObjectStore = deleteTransaction.objectStore("images");

    const deleteRequest = deleteObjectStore.delete(selectedImage.value);

    deleteRequest.onsuccess = (event) => {
      console.log("Data deleted successfully");
    };

    deleteRequest.onerror = (event) => {
      console.error("Error deleting data: ", event.target.error);
    };

    deleteTransaction.oncomplete = () => {
      console.log("Delete transaction completed");
      db.close();
    };
  };
}
</script>