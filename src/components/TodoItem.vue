/* eslint-disable no-console */
<template>
  <ul>
    <li v-for="todo in todos" v-bind:key="todo.id">
      <div v-if="todo.is_public" class="userInfoPublic">
        @{{ todo.user.name }}
      </div>
      <div class="view" v-if="type === 'private'">
          <div v-if="todo.is_completed" class="round">
            <input
              type="checkbox"
              id="todo.id"
              checked="true"
            />
            <label 
              v-on:click="handleTodoToggle(todo)"
              htmlFor="todo.id"></label>
          </div>
          <div v-else class="round">
            <input
              type="checkbox"
              id="todo.id"
            />
            <label 
              v-on:click="handleTodoToggle(todo)"
              htmlFor="todo.id"></label>
          </div>
      </div>
      <div class="labelContent">
          <strike class="todoLabel" v-if="todo.is_completed">
            <div>
              {{ todo.title }}
            </div>
          </strike>
          <div v-else>
            {{ todo.title }}
          </div>
      </div>
      <button v-if="type === 'private'" v-on:click="handleTodoDelete(todo)" class="closeBtn"> x </button>
    </li>
  </ul>
</template>

<script>
  import gql from 'graphql-tag'
  import { GET_MY_TODOS } from "./TodoPrivateList.vue"
  const TOGGLE_TODO = gql`
    mutation update_todos($id: Int!, $isCompleted: Boolean!) {
      update_todos(where: { id: { _eq: $id } }, _set: { is_completed: $isCompleted }) {
        affected_rows
      }
    }
  `
  export default {
    props: ['todos', 'type'],
    methods: {
      handleTodoToggle: function (todo) {
        // note: valiableというスペルミスでハマった..
        this.$apollo.mutate({
          mutation: TOGGLE_TODO,
          variables: {
            id: todo.id,
            isCompleted: !todo.is_completed
          },
          // ここでキャッシュをアップデートするが今回はDBのデータが正常に更新されることが前提の
          //「楽観的な更新」を実装する
          // ここのupdateの引数の受け取り方がいまいち..
          update: (store, { data: {update_todos } }) => {
            // Q.affected_rowsって何？
            // => graphQLのmutationのレスポンスに変更があったもののデータを含ませることができる。
            // *クエリを記述する際に返り値を指定できる。
            if (update_todos.affected_rows) {
              // eslint-disable-next-line no-console
              console.log(update_todos)
              if (this.type === 'private') {
                const data = store.readQuery({
                  query: GET_MY_TODOS,
                });
                const toggleTodo = data.todos.find(t => t.id === todo.id);
                // findの返り値は同じメモリを参照するみたいやね
                toggleTodo.is_completed = !todo.is_completed;
                store.writeQuery({
                  query: GET_MY_TODOS,
                  data
                })
              }
            }
          },
          // 楽観的なupdate(レスポンスが返ってくる前にUIを更新する。)
          // レスポンスが返ってきたらさらに自動で更新をかけてくれる。
          optimisticResponse: {
            __typename: 'Mutation',
            update_todos: {
              __typename: 'todos',
              id: todo.id,
              is_completed: !todo.is_completed,
              affected_rows: 1
            }
          }
        });
      },
      handleTodoDelete: function(todo) { // eslint-disable-line
        // delete todo from db
      }
    }
  }

</script>
