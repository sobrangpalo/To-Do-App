"use client";
import { useState, useEffect, useCallback, useMemo, createContext, useContext } from "react";

type Todo = { id: string; text: string; completed: boolean; createdAt: number; };
type Filter = "all" | "active" | "completed";

type ThemeContextType = { theme: "light"|"dark"; toggle: () => void };
const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

const ThemeProvider = ({ children }: { children: React.ReactNode }) => {
  const [theme, setTheme] = useState<"light"|"dark">("light");
  useEffect(()=>{ const saved=localStorage.getItem("todo_theme") as "light"|"dark"|null; if(saved) setTheme(saved); },[]);
  useEffect(()=>{ localStorage.setItem("todo_theme", theme); document.documentElement.classList.toggle("dark", theme==="dark"); },[theme]);
  const toggle = ()=>setTheme(t=>"light"===t?"dark":"light");
  return <ThemeContext.Provider value={{theme,toggle}}>{children}</ThemeContext.Provider>
};
const useTheme = ()=>{ const ctx = useContext(ThemeContext); if(!ctx) throw new Error("useTheme must be inside provider"); return ctx; };

const useLocalStorage = <T,>(key:string, initial:T)=>{
  const [state,setState] = useState<T>(()=>{ const raw=localStorage.getItem(key); return raw?JSON.parse(raw):initial; });
  useEffect(()=>{ localStorage.setItem(key, JSON.stringify(state)); },[state]);
  return [state,setState] as const;
};

export default function Home(){
  return <ThemeProvider><App /></ThemeProvider>;
}

const App = ()=>{
  const [todos,setTodos] = useLocalStorage<Todo[]>("todos_v1",[]);
  const [filter,setFilter] = useState<Filter>("all");
  const {theme,toggle} = useTheme();

  const addTodo = useCallback((text:string)=>{
    const newTodo:Todo={id:crypto.randomUUID(),text,completed:false,createdAt:Date.now()};
    setTodos(prev=>[newTodo,...prev]);
  },[]);

  const toggleTodo = useCallback((id:string)=>{
    setTodos(prev=>prev.map(t=>t.id===id?{...t,completed:!t.completed}:t));
  },[]);

  const deleteTodo = useCallback((id:string)=>setTodos(prev=>prev.filter(t=>t.id!==id)),[]);

  const clearCompleted = useCallback(()=>setTodos(prev=>prev.filter(t=>!t.completed)),[]);

  const filtered = useMemo(()=>{ 
    if(filter==="all") return todos; 
    if(filter==="active") return todos.filter(t=>!t.completed); 
    return todos.filter(t=>t.completed); 
  },[todos,filter]);

  return (
    <main className="min-h-screen flex justify-center bg-gray-50 dark:bg-gray-900 p-6">
      <div className="w-full max-w-2xl bg-white dark:bg-gray-800 shadow p-6 rounded-lg">
        <header className="flex justify-between items-center mb-4">
          <h1 className="text-xl font-bold text-gray-900 dark:text-gray-100">Todo App</h1>
          <button onClick={toggle} className="px-3 py-1 border rounded">{theme==="light"?"Dark Mode":"Light Mode"}</button>
        </header>

        <AddTodo onAdd={addTodo} />
        
        <div className="flex justify-between mt-4">
          <div className="flex gap-2">
            {["all","active","completed"].map(f=>(
              <button key={f} onClick={()=>setFilter(f as Filter)} className="px-3 py-1 rounded bg-gray-200 dark:bg-gray-700 capitalize">{f}</button>
            ))}
          </div>
          <button onClick={clearCompleted} className="text-red-500 text-sm">Clear completed</button>
        </div>

        <ul className="mt-4 divide-y divide-gray-200 dark:divide-gray-700">
          {filtered.length===0?<div className="text-center text-gray-500 py-5">No todos</div>:filtered.map(t=>(
            <TodoItem key={t.id} todo={t} onToggle={toggleTodo} onDelete={deleteTodo} />
          ))}
        </ul>
      </div>
    </main>
  );
};

const AddTodo = ({onAdd}:{onAdd:(text:string)=>void})=>{
  const [text,setText] = useState("");
  const submit = useCallback((e:React.FormEvent)=>{
    e.preventDefault();
    if(!text.trim()) return;
    onAdd(text.trim());
    setText("");
  },[text]);
  return (
    <form onSubmit={submit} className="flex gap-2">
      <input value={text} onChange={e=>setText(e.target.value)} placeholder="Add todo..." className="flex-1 border px-3 py-2 rounded"/>
      <button className="px-4 py-2 bg-indigo-600 text-white rounded">Add</button>
    </form>
  );
};

const TodoItem = React.memo(({todo,onToggle,onDelete}:{todo:Todo,onToggle:(id:string)=>void,onDelete:(id:string)=>void})=>{
  const handleToggle=useCallback(()=>onToggle(todo.id),[]);
  const handleDelete=useCallback(()=>onDelete(todo.id),[]);
  return (
    <li className="flex justify-between items-center py-3 px-2">
      <div className="flex gap-3">
        <input type="checkbox" checked={todo.completed} onChange={handleToggle}/>
        <span className={todo.completed?"line-through text-gray-400":""}>{todo.text}</span>
      </div>
      <button onClick={handleDelete} className="text-red-500 text-sm">Delete</button>
    </li>
  );
});
