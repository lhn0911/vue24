<template>
  <div class="container">
    <!-- Search and Add Post Button -->
    <div class="header">
      <div class="search-container">
        <input
          v-model="search"
          placeholder="Nhập từ khóa tìm kiếm"
          class="search-input"
        />
        <select class="filter-select">
          <option>Lọc bài viết</option>
        </select>
      </div>
      <button @click="openModal('add')" class="add-btn">
        Thêm mới bài viết
      </button>
    </div>

    <!-- Post Management Table -->
    <table class="post-table">
      <thead>
        <tr>
          <th>STT</th>
          <th>Tiêu đề</th>
          <th>Hình ảnh</th>
          <th>Ngày viết</th>
          <th>Trạng thái</th>
          <th>Chức năng</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(post, index) in filteredPosts" :key="post.id">
          <td>{{ index + 1 }}</td>
          <td>{{ post.title }}</td>
          <td><img :src="post.image" alt="Post Image" class="post-image" /></td>
          <td>{{ post.date }}</td>
          <td>
            <span
              :class="post.status === 'Đã xuất bản' ? 'published' : 'draft'"
              >{{ post.status }}</span
            >
          </td>
          <td>
            <button @click="confirmAction('block', post.id)" class="block-btn">
              Chặn
            </button>
            <button @click="openModal('edit', post)" class="edit-btn">
              Sửa
            </button>
            <button
              @click="confirmAction('delete', post.id)"
              class="delete-btn"
            >
              Xóa
            </button>
          </td>
        </tr>
      </tbody>
    </table>

    <!-- Add/Edit Post Modal -->
    <div v-if="showModal" class="modal">
      <div class="modal-content">
        <div class="close-btn">
          <button @click="showModal = false">×</button>
        </div>
        <h2>
          {{ modalMode === "edit" ? "Sửa bài viết" : "Thêm mới bài viết" }}
        </h2>
        <form @submit.prevent="handleSubmit">
          <label for="title">Tên bài viết</label><br />
          <input v-model="newPost.title" id="title" required /><br />
          <br />

          <label for="image">Hình ảnh</label><br />
          <input
            v-model="newPost.image"
            id="image"
            placeholder="URL hình ảnh"
          /><br />
          <br />

          <label for="content">Nội dung</label><br />
          <div class="rich-text-editor">
            <textarea
              v-model="newPost.content"
              id="content"
              rows="5"
              placeholder="Nhập nội dung"
            ></textarea>
          </div>
          <div class="modal-footer">
            <button type="button" @click="handleReset" class="reset-btn">
              Làm mới
            </button>
            <button type="submit" class="publish-btn">
              {{ modalMode === "edit" ? "Cập nhật" : "Xuất bản" }}
            </button>
          </div>
        </form>
      </div>
    </div>

    <!-- Confirmation Modal -->
    <div v-if="showConfirmModal" class="modal">
      <div class="modal-content">
        <h3>Xác nhận</h3>
        <p>{{ confirmMessage }}</p>
        <div class="modal-footer">
          <button @click="executeAction" class="confirm-btn">Xác nhận</button>
          <button @click="showConfirmModal = false" class="cancel-btn">
            Hủy
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";
import axios from "axios";

const search = ref("");
const showModal = ref(false);
const modalMode = ref("add"); // Chế độ 'add' hoặc 'edit'
const currentPostId = ref(null); // Lưu lại ID của bài viết đang sửa
const newPost = ref({
  title: "",
  image: "",
  content: "",
  status: "Đã xuất bản",
  date: new Date().toLocaleDateString(),
});

const posts = ref([]);
const showConfirmModal = ref(false);
const confirmMessage = ref("");
let actionType = ref("");
let actionPostId = ref(null);

// Lọc bài viết theo từ khóa tìm kiếm
const filteredPosts = computed(() => {
  return Array.isArray(posts.value)
    ? posts.value.filter((post) =>
        post.title.toLowerCase().includes(search.value.toLowerCase())
      )
    : [];
});

// Lấy dữ liệu từ database
const fetchPosts = async () => {
  try {
    const response = await axios.get("http://localhost:8080/post");
    posts.value = response.data;
  } catch (error) {
    console.error("Lỗi khi lấy dữ liệu bài viết:", error);
  }
};

onMounted(() => {
  fetchPosts();
});

// Mở modal thêm hoặc sửa
const openModal = (mode, post = null) => {
  modalMode.value = mode;
  if (mode === "edit" && post) {
    currentPostId.value = post.id;
    newPost.value = { ...post };
  } else {
    handleReset();
  }
  showModal.value = true;
};

// Thêm hoặc cập nhật bài viết
const handleSubmit = async () => {
  if (newPost.value.title && newPost.value.content) {
    try {
      const response = await axios.post("http://localhost:8080/post", {
        ...newPost.value,
        id: Math.max(...posts.value.map((post) => post.id)) + 1, // Tạo id mới
      });

      if (response.status === 201) {
        // Nếu lưu thành công, thêm bài viết vào danh sách
        posts.value.push(response.data);
        showModal.value = false;
        handleReset();
      } else {
        console.error("Không thể thêm bài viết:", response.status);
      }
    } catch (error) {
      console.error("Lỗi khi lưu bài viết:", error);
    }
  }
};

// Xác nhận hành động
const confirmAction = (type, id) => {
  actionType.value = type;
  actionPostId.value = id;
  if (type === "delete") {
    confirmMessage.value = "Bạn có chắc muốn xóa bài viết này?";
  } else if (type === "block") {
    confirmMessage.value = "Bạn có chắc muốn chặn bài viết này?";
  }
  showConfirmModal.value = true;
};

// Thực hiện hành động
const executeAction = async () => {
  if (actionType.value === "delete") {
    deletePost(actionPostId.value);
  } else if (actionType.value === "block") {
    blockPost(actionPostId.value);
  }
  showConfirmModal.value = false;
};

// Xóa bài viết
const deletePost = async (id) => {
  try {
    await axios.delete(`http://localhost:8080/post/${id}`);
    posts.value = posts.value.filter((post) => post.id !== id);
  } catch (error) {
    console.error("Lỗi khi xóa bài viết:", error);
  }
};

// Làm mới dữ liệu khi thêm bài viết
const handleReset = () => {
  newPost.value = {
    title: "",
    image: "",
    content: "",
    status: "Đã xuất bản",
    date: new Date().toLocaleDateString(),
  };
};

// Chặn bài viết
const blockPost = (id) => {
  console.log("Chặn bài viết với ID:", id);
};
</script>

<style scoped>
.container {
  font-family: Arial, sans-serif;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 20px;
}

.search-container {
  display: flex;
  gap: 10px;
}

.search-input,
.filter-select {
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.search-input {
  width: 300px;
}

.filter-select {
  width: 150px;
}

.add-btn {
  padding: 8px 16px;
  background-color: #1a73e8;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.post-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 20px;
}

.post-table th,
.post-table td {
  border: 1px solid #e0e0e0;
  padding: 12px;
  text-align: left;
}

.post-table th {
  background-color: #f8f9fa;
  font-weight: bold;
}

.post-image {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  object-fit: cover;
}

.published {
  background-color: #defec8;
  color: #07892c;
  padding: 4px 8px;
  border-radius: 5px;
  font-size: 15px;
  border: 1px solid black;
}

.draft {
  background-color: #fce8e6;
  color: #d93025;
  padding: 4px 8px;
  border-radius: 12px;
  font-size: 12px;
}
.block-btn,
.edit-btn,
.delete-btn {
  padding: 6px 12px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  margin-right: 5px;
}

.block-btn {
  background-color: #fbbc04;
  color: white;
}

.edit-btn {
  background-color: #1a73e8;
  color: white;
}

.delete-btn {
  background-color: #ea4335;
  color: white;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background: white;
  padding: 20px;
  border-radius: 4px;
  width: 400px;
  position: relative;
}

.close-btn {
  display: flex;
  justify-content: end;
}

.modal-footer {
  display: flex;
  justify-content: space-between;
  margin-top: 20px;
}

.reset-btn {
  background-color: #f1c40f;
  border: none;
  color: white;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
}

.publish-btn {
  background-color: #28a745;
  border: none;
  color: white;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
}
.confirm-btn {
  background-color: #28a745;
  border: none;
  color: white;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
}

.cancel-btn {
  background-color: #f1c40f;
  border: none;
  color: white;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
}
</style>
