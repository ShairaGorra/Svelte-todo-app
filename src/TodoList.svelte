<script>
  export let todos = [];
  export let onToggle;
  export let onDelete;
  export let onUpdate;
  export let onAddSubItem;
  export let onToggleSubItem;
  export let onDeleteSubItem;
  export let searchQuery = '';
  export let selectedCategory = 'all';

  let editingId = null;
  let editText = '';
  let currentTab = 'todo';
  let subItemTexts = {};
  
  // Track collapsed status for each grocery list ID
  let collapsedGroceries = {};

  function toggleCollapse(id) {
    collapsedGroceries[id] = !collapsedGroceries[id];
  }

  function startEdit(todo) {
    editingId = todo.id;
    editText = todo.text;
  }

  function saveEdit(id) {
    if (editText.trim() !== '') {
      onUpdate(id, editText);
    }
    editingId = null;
    editText = '';
  }

  function handleAddSub(todoId) {
    const text = subItemTexts[todoId];
    if (text && text.trim() !== '') {
      onAddSubItem(todoId, text.trim());
      subItemTexts[todoId] = '';
    }
  }

  $: filteredTodos = todos.filter(t => {
    const matchesSearch = t.text.toLowerCase().includes(searchQuery.toLowerCase());
    if (selectedCategory === 'grocery') {
      return matchesSearch && t.isGrocery;
    } else if (selectedCategory === 'task') {
      return matchesSearch && !t.isGrocery;
    }
    return matchesSearch;
  });

  $: activeTodos = filteredTodos.filter(t => !t.completed);
  $: completedTodos = filteredTodos.filter(t => t.completed);
</script>

<div class="cafe-container">
  <div class="cafe-tabs">
    <button 
      class="cafe-tab" 
      class:active={currentTab === 'todo'} 
      on:click={() => currentTab = 'todo'}
    >
      To-Do ({activeTodos.length})
    </button>
    <button 
      class="cafe-tab" 
      class:active={currentTab === 'completed'} 
      on:click={() => currentTab = 'completed'}
    >
      Completed ({completedTodos.length})
    </button>
  </div>

  {#if currentTab === 'todo'}
    <div class="cafe-shelf">
      {#if activeTodos.length === 0}
        <p class="empty">No tasks found.</p>
      {:else}
        <ul>
          {#each activeTodos as todo (todo.id)}
            <li class="cafe-card">
              <div class="card-main">
                <input 
                  type="checkbox" 
                  checked={todo.completed} 
                  on:change={() => onToggle(todo.id)} 
                />

                {#if editingId === todo.id}
                  <input 
                    type="text" 
                    bind:value={editText} 
                    on:keydown={(e) => e.key === 'Enter' && saveEdit(todo.id)}
                  />
                  <button class="cafe-btn save" on:click={() => saveEdit(todo.id)}>Save</button>
                {:else}
                  <span class="task-text">{todo.text}</span>
                  {#if todo.isGrocery}
                    <div class="badge-group">
                      <span class="grocery-badge">🛒 Grocery</span>
                      <button class="collapse-btn" on:click={() => toggleCollapse(todo.id)}>
                        {collapsedGroceries[todo.id] ? '▶ Show Items' : '▼ Hide Items'}
                      </button>
                    </div>
                  {/if}
                  <div class="card-actions">
                    <button class="cafe-btn" on:click={() => startEdit(todo)}>Edit</button>
                    <button class="cafe-btn delete" on:click={() => onDelete(todo.id)}>Delete</button>
                  </div>
                {/if}
              </div>

              <!-- Nested Sub-checklist for Groceries (collapsible) -->
              {#if todo.isGrocery && !collapsedGroceries[todo.id]}
                <div class="sub-checklist">
                  <div class="sub-header">Grocery Items:</div>
                  {#if todo.subItems && todo.subItems.length > 0}
                    <ul class="sub-list">
                      {#each todo.subItems as sub (sub.id)}
                        <li class="sub-item" class:completed-sub={sub.completed}>
                          <input 
                            type="checkbox" 
                            checked={sub.completed} 
                            on:change={() => onToggleSubItem(todo.id, sub.id)}
                          />
                          <span class="sub-text">{sub.text}</span>
                          <button class="sub-del" on:click={() => onDeleteSubItem(todo.id, sub.id)}>×</button>
                        </li>
                      {/each}
                    </ul>
                  {/if}
                  <div class="sub-input-group">
                    <input 
                      type="text" 
                      placeholder="Add item..." 
                      bind:value={subItemTexts[todo.id]}
                      on:keydown={(e) => e.key === 'Enter' && handleAddSub(todo.id)}
                    />
                    <button class="sub-add-btn" on:click={() => handleAddSub(todo.id)}>+</button>
                  </div>
                </div>
              {/if}
            </li>
          {/each}
        </ul>
      {/if}
    </div>
  {/if}

  {#if currentTab === 'completed'}
    <div class="cafe-shelf">
      {#if completedTodos.length === 0}
        <p class="empty">No completed tasks found.</p>
      {:else}
        <ul>
          {#each completedTodos as todo (todo.id)}
            <li class="cafe-card completed">
              <div class="card-main">
                <input 
                  type="checkbox" 
                  checked={todo.completed} 
                  on:change={() => onToggle(todo.id)} 
                />
                <span class="task-text">{todo.text}</span>
                {#if todo.isGrocery}
                  <div class="badge-group">
                    <span class="grocery-badge">🛒 Grocery</span>
                    <button class="collapse-btn" on:click={() => toggleCollapse(todo.id)}>
                      {collapsedGroceries[todo.id] ? '▶ Show Items' : '▼ Hide Items'}
                    </button>
                  </div>
                {/if}
                <button class="cafe-btn delete" on:click={() => onDelete(todo.id)}>Delete</button>
              </div>

              {#if todo.isGrocery && !collapsedGroceries[todo.id] && todo.subItems && todo.subItems.length > 0}
                <div class="sub-checklist">
                  <ul class="sub-list">
                    {#each todo.subItems as sub (sub.id)}
                      <li class="sub-item" class:completed-sub={sub.completed}>
                        <input 
                          type="checkbox" 
                          checked={sub.completed} 
                          on:change={() => onToggleSubItem(todo.id, sub.id)}
                        />
                        <span class="sub-text">{sub.text}</span>
                      </li>
                    {/each}
                  </ul>
                </div>
              {/if}
            </li>
          {/each}
        </ul>
      {/if}
    </div>
  {/if}
</div>

<style>
  .cafe-container {
    flex: 1;
    display: flex;
    flex-direction: column;
    margin-top: 15px;
    overflow: hidden;
  }

  .cafe-tabs {
    display: flex;
    gap: 10px;
    margin-bottom: 15px;
  }

  .cafe-tab {
    flex: 1;
    background: #3b2c20;
    border: 1px solid #5a4231;
    color: #d1bfa7;
    padding: 10px;
    border-radius: 8px;
    cursor: pointer;
    font-size: 0.9rem;
    font-family: serif;
    transition: all 0.2s;
  }

  .cafe-tab.active {
    background: #5a4231;
    border-color: #d4a373;
    color: #faedcd;
    font-weight: bold;
  }

  .cafe-shelf {
    flex: 1;
    background: #281d15;
    border: 1px solid #483525;
    padding: 18px;
    padding-bottom: 60px;
    border-radius: 10px;
    overflow-y: auto;
    box-shadow: inset 0 4px 12px rgba(0, 0, 0, 0.4);
  }

  ul {
    list-style: none;
    padding: 0;
    margin: 0;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .cafe-card {
    background: #33241b;
    border: 1px solid #523b2b;
    padding: 14px 16px;
    border-radius: 8px;
    color: #faedcd;
    box-shadow: 0 2px 6px rgba(0,0,0,0.2);
  }

  .card-main {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .cafe-card.completed {
    opacity: 0.6;
    background: #251a13;
  }

  .cafe-card.completed .task-text {
    text-decoration: line-through;
    color: #a39179;
  }

  .task-text {
    flex: 1;
    font-family: serif;
    font-size: 1.05rem;
    word-break: break-all;
    color: #fefae0;
  }

  .badge-group {
    display: flex;
    align-items: center;
    gap: 6px;
  }

  .grocery-badge {
    background: #483525;
    color: #d4a373;
    border: 1px solid #d4a373;
    padding: 3px 10px;
    border-radius: 12px;
    font-size: 0.75rem;
    font-family: serif;
  }

  .collapse-btn {
    background: #1c130e;
    border: 1px solid #483525;
    color: #d4a373;
    padding: 3px 8px;
    border-radius: 12px;
    font-size: 0.75rem;
    font-family: serif;
    cursor: pointer;
    transition: all 0.2s;
  }

  .collapse-btn:hover {
    background: #483525;
    color: #faedcd;
  }

  .card-actions {
    display: flex;
    gap: 6px;
  }

  input[type="checkbox"] {
    cursor: pointer;
    width: 18px;
    height: 18px;
    accent-color: #d4a373;
  }

  input[type="text"] {
    flex: 1;
    background: #1c130e;
    border: 1px solid #d4a373;
    color: #faedcd;
    padding: 6px 10px;
    border-radius: 4px;
    outline: none;
    font-family: serif;
  }

  .cafe-btn {
    background: transparent;
    border: 1px solid #795d44;
    color: #d1bfa7;
    padding: 6px 12px;
    border-radius: 4px;
    cursor: pointer;
    font-size: 0.85rem;
    font-family: serif;
  }

  .cafe-btn:hover {
    background: #483525;
    color: #faedcd;
  }

  .cafe-btn.save {
    border-color: #d4a373;
    color: #d4a373;
  }

  .cafe-btn.delete:hover {
    border-color: #bc6c25;
    color: #bc6c25;
  }

  .sub-checklist {
    margin-top: 12px;
    padding-top: 10px;
    border-top: 1px solid #483525;
    margin-left: 30px;
  }

  .sub-header {
    font-size: 0.8rem;
    color: #d4a373;
    margin-bottom: 6px;
    font-family: serif;
  }

  .sub-list {
    display: flex;
    flex-direction: column;
    gap: 6px;
    margin-bottom: 8px;
  }

  .sub-item {
    display: flex;
    align-items: center;
    gap: 8px;
    background: #1c130e;
    padding: 6px 10px;
    border-radius: 4px;
    border: 1px solid #483525;
  }

  .completed-sub .sub-text {
    text-decoration: line-through;
    color: #a39179;
  }

  .sub-text {
    flex: 1;
    font-size: 0.9rem;
    font-family: serif;
    color: #faedcd;
  }

  .sub-del {
    background: transparent;
    border: none;
    color: #bc6c25;
    cursor: pointer;
    font-size: 1rem;
    padding: 0 4px;
  }

  .sub-input-group {
    display: flex;
    gap: 6px;
  }

  .sub-input-group input {
    background: #1c130e;
    border: 1px solid #483525;
    color: #faedcd;
    padding: 5px 8px;
    font-size: 0.85rem;
    border-radius: 4px;
  }

  .sub-add-btn {
    background: #d4a373;
    color: #281d15;
    border: none;
    font-weight: bold;
    padding: 5px 12px;
    border-radius: 4px;
    cursor: pointer;
  }

  .empty {
    color: #a39179;
    font-size: 1rem;
    text-align: center;
    margin-top: 50px;
    font-family: serif;
    font-style: italic;
  }
</style>