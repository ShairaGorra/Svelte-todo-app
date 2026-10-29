<script>
  import { onMount } from 'svelte';
  import { supabase } from './supabaseClient';
  import TodoList from './TodoList.svelte';

  let todos = [];
  let searchQuery = '';
  let selectedCategory = 'all';
  let newText = '';
  let isGrocery = false;
  let showModal = false;

  onMount(async () => {
    await fetchTodos();
  });

  async function fetchTodos() {
    const { data, error } = await supabase
      .from('todos')
      .select('*, sub_todos(*)')
      .order('id', { ascending: true });

    if (error) {
      console.error('Error fetching todos:', error);
    } else {
      todos = data.map(t => ({
        id: t.id,
        text: t.text,
        completed: t.completed,
        isGrocery: t.is_grocery,
        subItems: (t.sub_todos || []).map(s => ({
          id: s.id,
          text: s.text,
          completed: s.completed
        }))
      }));
    }
  }

  async function addTodo() {
    if (newText.trim() === '') return;

    const { data, error } = await supabase
      .from('todos')
      .insert([{ text: newText, completed: false, is_grocery: isGrocery }])
      .select();

    if (error) {
      console.error('Error adding todo:', error);
    } else if (data && data[0]) {
      const newTodo = {
        id: data[0].id,
        text: data[0].text,
        completed: data[0].completed,
        isGrocery: data[0].is_grocery,
        subItems: []
      };
      todos = [...todos, newTodo];
    }

    newText = '';
    isGrocery = false;
    showModal = false;
  }

  async function toggleTodo(id) {
    const target = todos.find(t => t.id === id);
    if (!target) return;
    const newStatus = !target.completed;

    const { error } = await supabase
      .from('todos')
      .update({ completed: newStatus })
      .eq('id', id);

    if (error) {
      console.error('Error toggling todo:', error);
    } else {
      todos = todos.map(todo => todo.id === id ? { ...todo, completed: newStatus } : todo);
    }
  }

  async function deleteTodo(id) {
    const confirmed = window.confirm("Are you sure you want to delete this task?");
    if (!confirmed) return;

    const { error } = await supabase
      .from('todos')
      .delete()
      .eq('id', id);

    if (error) {
      console.error('Error deleting todo:', error);
    } else {
      todos = todos.filter(todo => todo.id !== id);
    }
  }

  async function updateTodo(id, updatedText) {
    const { error } = await supabase
      .from('todos')
      .update({ text: updatedText })
      .eq('id', id);

    if (error) {
      console.error('Error updating todo:', error);
    } else {
      todos = todos.map(todo => todo.id === id ? { ...todo, text: updatedText } : todo);
    }
  }

  async function addSubItem(todoId, text) {
    const { data, error } = await supabase
      .from('sub_todos')
      .insert([{ todo_id: todoId, text: text, completed: false }])
      .select();

    if (error) {
      console.error('Error adding sub-item:', error);
    } else if (data && data[0]) {
      todos = todos.map(todo => {
        if (todo.id === todoId) {
          return {
            ...todo,
            subItems: [...todo.subItems, { id: data[0].id, text: data[0].text, completed: data[0].completed }]
          };
        }
        return todo;
      });
    }
  }

  async function toggleSubItem(todoId, subId) {
    const targetTodo = todos.find(t => t.id === todoId);
    if (!targetTodo) return;
    const targetSub = targetTodo.subItems.find(s => s.id === subId);
    if (!targetSub) return;
    const newStatus = !targetSub.completed;

    const { error } = await supabase
      .from('sub_todos')
      .update({ completed: newStatus })
      .eq('id', subId);

    if (error) {
      console.error('Error toggling sub-item:', error);
    } else {
      todos = todos.map(todo => {
        if (todo.id === todoId) {
          return {
            ...todo,
            subItems: todo.subItems.map(sub => sub.id === subId ? { ...sub, completed: newStatus } : sub)
          };
        }
        return todo;
      });
    }
  }

  async function deleteSubItem(todoId, subId) {
    const confirmed = window.confirm("Are you sure you want to delete this grocery item?");
    if (!confirmed) return;

    const { error } = await supabase
      .from('sub_todos')
      .delete()
      .eq('id', subId);

    if (error) {
      console.error('Error deleting sub-item:', error);
    } else {
      todos = todos.map(todo => {
        if (todo.id === todoId) {
          return {
            ...todo,
            subItems: todo.subItems.filter(sub => sub.id !== subId)
          };
        }
        return todo;
      });
    }
  }

  $: remainingCount = todos.filter(t => !t.completed).length;
</script>

<main>
  <div class="wood-board">
    <div class="header-row">
      <h2>My To-Do List</h2>
      <div class="header-right">
        <span class="counter">Tasks: {remainingCount}</span>
        <button class="add-task-btn" on:click={() => showModal = true}>
          <span class="plus-icon">+</span> New Task
        </button>
      </div>
    </div>

    <div class="filter-group">
      <input 
        type="text" 
        bind:value={searchQuery} 
        placeholder="Search tasks..." 
      />
      <select bind:value={selectedCategory} class="category-select">
        <option value="all">All Categories</option>
        <option value="task">Tasks Only</option>
        <option value="grocery">Groceries Only</option>
      </select>
    </div>

    <TodoList 
      {todos} 
      {searchQuery}
      {selectedCategory}
      onToggle={toggleTodo} 
      onDelete={deleteTodo} 
      onUpdate={updateTodo}
      onAddSubItem={addSubItem}
      onToggleSubItem={toggleSubItem}
      onDeleteSubItem={deleteSubItem}
    />
  </div>

  {#if showModal}
    <div class="modal-backdrop">
      <div class="modal">
        <h3>Add New Task</h3>
        <input 
          type="text" 
          bind:value={newText} 
          placeholder="Type task description..." 
          on:keydown={(e) => e.key === 'Enter' && addTodo()}
          autofocus
        />
        
        <label class="checkbox-label">
          <input type="checkbox" bind:checked={isGrocery} />
          <span>Is this a grocery list?</span>
        </label>

        <div class="modal-btns">
          <button class="cancel-btn" on:click={() => showModal = false}>Cancel</button>
          <button class="add-btn" on:click={addTodo}>Add Task</button>
        </div>
      </div>
    </div>
  {/if}
</main>

<style>
  :global(html, body) {
    margin: 0;
    padding: 0;
    min-height: 100vh;
    background-color: #1a1412;
    color: #f1e4d3;
    font-family: 'Georgia', 'Times New Roman', serif;
    box-sizing: border-box;
  }

  main {
    min-height: 100vh;
    width: 100%;
    display: flex;
    justify-content: center;
    align-items: flex-start;
    padding: 30px 16px;
    box-sizing: border-box;
    background: radial-gradient(circle at center, #2e231d 0%, #17110e 100%);
  }

  .wood-board {
    width: 100%;
    max-width: 720px;
    background: #2b201a;
    border: 1px solid #4a382d;
    border-top: 3px solid #6b5242;
    border-radius: 12px;
    padding: 24px;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.6), inset 0 1px 1px rgba(255, 255, 255, 0.05);
    position: relative;
    box-sizing: border-box;
  }

  .header-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;
    border-bottom: 1px solid #4a382d;
    padding-bottom: 14px;
  }

  .header-right {
    display: flex;
    align-items: flex-end;
    gap: 12px;
  }

  h2 {
    color: #e2b887;
    margin: 0;
    font-size: 1.8rem;
    font-weight: normal;
    letter-spacing: 0.5px;
  }

  .counter {
    background: #1f1713;
    border: 1px solid #523e32;
    color: #e2b887;
    padding: 0 14px;
    height: 32px;
    border-radius: 16px;
    font-size: 0.82rem;
    font-family: serif;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    box-sizing: border-box;
    margin-bottom: 3px; 
  }

  .add-task-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    background: linear-gradient(180deg, #e0b078 0%, #bd8c54 100%);
    color: #1a130f;
    border: 1px solid #fce3c7;
    border-radius: 6px;
    padding: 0 16px;
    height: 38px;
    font-size: 0.88rem;
    font-family: serif;
    font-weight: bold;
    cursor: pointer;
    box-shadow: 0 0 12px rgba(224, 176, 120, 0.35), 0 3px 6px rgba(0, 0, 0, 0.4), inset 0 1px 0 rgba(255, 255, 255, 0.5);
    transition: all 0.2s ease-in-out;
    box-sizing: border-box;
  }

  .plus-icon {
    font-size: 1.1rem;
    line-height: 1;
    font-weight: bold;
  }

  .add-task-btn:hover {
    background: linear-gradient(180deg, #f0c38c 0%, #cc995e 100%);
    box-shadow: 0 0 18px rgba(240, 195, 140, 0.55), 0 4px 10px rgba(0, 0, 0, 0.5), inset 0 1px 0 rgba(255, 255, 255, 0.7);
    transform: translateY(-1px);
    border-color: #ffffff;
  }

  .add-task-btn:active {
    transform: translateY(1px);
    box-shadow: 0 0 8px rgba(224, 176, 120, 0.3), 0 1px 3px rgba(0, 0, 0, 0.6);
  }

  .filter-group {
    display: flex;
    gap: 10px;
    margin-bottom: 16px;
  }

  .filter-group input {
    flex: 1;
    background: #1a130f;
    border: 1px solid #4a382d;
    color: #f1e4d3;
    padding: 10px 14px;
    border-radius: 6px;
    outline: none;
    font-size: 0.95rem;
    font-family: serif;
  }

  .filter-group input:focus {
    border-color: #c99a64;
  }

  .filter-group input::placeholder {
    color: #8c7362;
  }

  .category-select {
    background: #1a130f;
    border: 1px solid #4a382d;
    color: #d8a873;
    padding: 0 12px;
    border-radius: 6px;
    outline: none;
    font-size: 0.9rem;
    font-family: serif;
    cursor: pointer;
  }

  .category-select:focus {
    border-color: #c99a64;
  }

  .modal-backdrop {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background: rgba(18, 13, 10, 0.85);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 100;
  }

  .modal {
    background: #2b201a;
    border: 1px solid #c99a64;
    padding: 26px;
    border-radius: 10px;
    width: 90%;
    max-width: 380px;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.7);
  }

  .modal h3 {
    color: #e2b887;
    margin-top: 0;
    margin-bottom: 16px;
    font-weight: normal;
    font-size: 1.25rem;
  }

  .modal input[type="text"] {
    width: 100%;
    box-sizing: border-box;
    background: #1a130f;
    border: 1px solid #4a382d;
    color: #f1e4d3;
    padding: 10px 12px;
    border-radius: 6px;
    outline: none;
    font-size: 0.95rem;
    font-family: serif;
    margin-bottom: 16px;
  }

  .modal input[type="text"]:focus {
    border-color: #c99a64;
  }

  .checkbox-label {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 20px;
    cursor: pointer;
    font-size: 0.9rem;
    color: #cbb29c;
  }

  .checkbox-label input {
    cursor: pointer;
    width: 16px;
    height: 16px;
    accent-color: #c99a64;
  }

  .modal-btns {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
  }

  .cancel-btn {
    background: transparent;
    border: 1px solid #523e32;
    color: #cbb29c;
    padding: 7px 14px;
    border-radius: 5px;
    cursor: pointer;
    font-family: serif;
  }

  .add-btn {
    background: #c99a64;
    color: #1a130f;
    border: none;
    padding: 7px 18px;
    border-radius: 5px;
    font-weight: bold;
    cursor: pointer;
    font-family: serif;
  }

  .add-btn:hover {
    background: #e2b887;
  }
</style>