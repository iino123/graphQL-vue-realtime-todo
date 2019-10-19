<template>
  <div class="formInput">
    <input 
	class="input" 
	placeholder="What needs to be done?" 
	v-model="newTodo"
	@keyup.enter="addTodo"
    />
    <i class="downArrow fa fa-angle-right" />
  </div>
</template>

<script>
  import gql from "graphql-tag";
  import { GET_MY_TODOS } from "./TodoPrivateList.vue";
  const ADD_TODO = gql`
    mutation insert_todos($todo: String!, $isPublic: Boolean!) {
      insert_todos(objects: {title: $todo, is_public: $isPublic}) {
        affected_rows
        returning {
          id
          title
          is_completed
          created_at
          is_public
        }
      }
    }
  `;
  export default {
    props: ['type'],
    data() {
      return {
        newTodo: '',
      }
    },
    methods: {
      addTodo: function () {
        const title = this.newTodo && this.newTodo.trim()
        const isPublic = this.type === "public" ? true : false;
        this.$apollo.mutate({
          mutation: ADD_TODO,
          variables: {
            todo: title,
            isPublic: isPublic
          },
          // UIを更新する為に再びapiをコールする必要はない
          // ADD_TODOの返り値にADD_TODOしたもの自体が返るようにしておいて、その値をキャッシュに書き込む。
          // 今回はGET_MY_TODOSによるUIを更新したい。
          update: (cache, { data: { insert_todos } }) => {
            // cache => cache
            // data => mutationの結果
            try {
              if (this.type === "private") {
                // readQueryを使うとサーバにデータを送信しない
                const data = cache.readQuery({
                  query: GET_MY_TODOS
                });
                const insertedTodo = insert_todos.returning;
                data.todos.splice(0, 0, insertedTodo[0]);
                // UIを更新する為にローカルキャッシュの値を変更する(サーバには値は送信されない)
                cache.writeQuery({
                  query: GET_MY_TODOS,
                  data
                });
              }
            } catch (e) {
              console.error(e);
            }
          },
        });
        // 入力部分のUIをリセットする
        this.newTodo = '';
      },
    }
  }
</script>
