# process-simulation2
process schedulling simulator 
# Process Scheduling Simulator
# Mini Project - Operating Systems
# Algorithms: FCFS, SJF (Non-Preemptive), SJF (Preemptive), Round Robin

import tkinter as tk
from tkinter import ttk, messagebox
import copy


# Process class to store process details
class Process:
    def __init__(self, pid, arrival, burst):
        self.pid = pid
        self.arrival = arrival
        self.burst = burst
        self.remaining = burst
        self.completion = 0
        self.turnaround = 0
        self.waiting = 0


# Gantt chart block
class GanttBlock:
    def __init__(self, pid, start, end):
        self.pid = pid
        self.start = start
        self.end = end


# --------- Scheduling Algorithms ---------

def fcfs(processes):
    procs = sorted(copy.deepcopy(processes), key=lambda p: (p.arrival, p.pid))
    gantt = []
    time = 0

    for p in procs:
        if time < p.arrival:
            gantt.append(GanttBlock("Idle", time, p.arrival))
            time = p.arrival
        gantt.append(GanttBlock(p.pid, time, time + p.burst))
        time += p.burst
        p.completion = time
        p.turnaround = p.completion - p.arrival
        p.waiting = p.turnaround - p.burst

    return procs, gantt


def sjf(processes):
    procs = copy.deepcopy(processes)
    remaining = list(procs)
    finished = []
    gantt = []
    time = 0

    while remaining:
        available = [p for p in remaining if p.arrival <= time]
        if not available:
            next_arrival = min(p.arrival for p in remaining)
            gantt.append(GanttBlock("Idle", time, next_arrival))
            time = next_arrival
            continue

        # pick process with shortest burst time
        chosen = min(available, key=lambda p: (p.burst, p.arrival, p.pid))
        gantt.append(GanttBlock(chosen.pid, time, time + chosen.burst))
        time += chosen.burst
        chosen.completion = time
        chosen.turnaround = chosen.completion - chosen.arrival
        chosen.waiting = chosen.turnaround - chosen.burst
        remaining.remove(chosen)
        finished.append(chosen)

    return finished, gantt


def sjf_preemptive(processes):
    procs = copy.deepcopy(processes)
    n = len(procs)
    gantt = []
    time = 0
    completed = 0
    prev_pid = None

    while completed < n:
        available = [p for p in procs if p.arrival <= time and p.remaining > 0]
        if not available:
            next_arrival = min(p.arrival for p in procs if p.remaining > 0)
            if prev_pid != "Idle":
                gantt.append(GanttBlock("Idle", time, next_arrival))
            else:
                gantt[-1].end = next_arrival
            prev_pid = "Idle"
            time = next_arrival
            continue

        # pick process with shortest remaining time
        chosen = min(available, key=lambda p: (p.remaining, p.arrival, p.pid))

        # merge with previous block if same process
        if prev_pid == chosen.pid:
            gantt[-1].end = time + 1
        else:
            gantt.append(GanttBlock(chosen.pid, time, time + 1))

        prev_pid = chosen.pid
        chosen.remaining -= 1
        time += 1

        if chosen.remaining == 0:
            chosen.completion = time
            chosen.turnaround = chosen.completion - chosen.arrival
            chosen.waiting = chosen.turnaround - chosen.burst
            completed += 1

    return procs, gantt


def round_robin(processes, quantum):
    procs = copy.deepcopy(processes)
    procs.sort(key=lambda p: (p.arrival, p.pid))
    gantt = []
    queue = []
    remaining = list(procs)
    time = 0

    # add processes that arrive at time 0
    for p in remaining[:]:
        if p.arrival <= time:
            queue.append(p)
            remaining.remove(p)

    while queue or remaining:
        if not queue:
            next_arrival = min(p.arrival for p in remaining)
            gantt.append(GanttBlock("Idle", time, next_arrival))
            time = next_arrival
            for p in remaining[:]:
                if p.arrival <= time:
                    queue.append(p)
                    remaining.remove(p)
            continue

        current = queue.pop(0)
        run_time = min(quantum, current.remaining)
        gantt.append(GanttBlock(current.pid, time, time + run_time))
        time += run_time
        current.remaining -= run_time

        # add newly arrived processes before re-queuing current
        for p in remaining[:]:
            if p.arrival <= time:
                queue.append(p)
                remaining.remove(p)

        if current.remaining > 0:
            queue.append(current)
        else:
            current.completion = time
            current.turnaround = current.completion - current.arrival
            current.waiting = current.turnaround - current.burst

    return procs, gantt


# colors for gantt chart
COLORS = ["#4FC3F7", "#81C784", "#FFB74D", "#E57373", "#BA68C8",
           "#4DD0E1", "#FFD54F", "#AED581", "#F06292", "#7986CB"]
IDLE_COLOR = "#E0E0E0"

# sample examples for each algorithm
EXAMPLES = {
    "FCFS": [
        Process("P1", 0, 4),
        Process("P2", 1, 3),
        Process("P3", 2, 1),
        Process("P4", 3, 5),
    ],
    "SJF": [
        Process("P1", 0, 7),
        Process("P2", 2, 4),
        Process("P3", 4, 1),
        Process("P4", 5, 4),
    ],
    "SJF Preemptive": [
        Process("P1", 0, 6),
        Process("P2", 1, 2),
        Process("P3", 3, 4),
        Process("P4", 5, 1),
    ],
    "Round Robin": [
        Process("P1", 0, 5),
        Process("P2", 1, 3),
        Process("P3", 2, 8),
        Process("P4", 3, 6),
    ],
}


# --------- Main Application ---------

class SchedulerApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("CPU Scheduling Simulator")
        self.geometry("960x700")
        self.configure(bg="#f0f0f0")

        self.processes = []
        self.color_map = {}

        self.build_ui()

    def build_ui(self):
        style = ttk.Style(self)
        style.theme_use("clam")
        style.configure("TLabelframe.Label", foreground="black", font=("Arial", 10, "bold"))
        style.configure("TLabelframe", background="#f0f0f0")
        style.configure("Blue.TButton", foreground="white", background="#1976D2", font=("Arial", 9, "bold"))
        style.map("Blue.TButton", foreground=[("active", "white")], background=[("active", "#1565C0")])
        style.configure("Green.TButton", foreground="white", background="#388E3C", font=("Arial", 10, "bold"))
        style.map("Green.TButton", foreground=[("active", "white")], background=[("active", "#2E7D32")])
        style.configure("Red.TButton", foreground="white", background="#D32F2F", font=("Arial", 10, "bold"))
        style.map("Red.TButton", foreground=[("active", "white")], background=[("active", "#C62828")])
        style.configure("Orange.TButton", foreground="white", background="#F57C00", font=("Arial", 9, "bold"))
        style.map("Orange.TButton", foreground=[("active", "white")], background=[("active", "#E65100")])

        # title label
        title = tk.Label(self, text="CPU Scheduling Simulator", font=("Arial", 14, "bold"),
                         bg="#37474F", fg="white", pady=8)
        title.pack(fill="x")

        # main frame - left and right
        main = tk.Frame(self, bg="#f0f0f0")
        main.pack(fill="both", expand=True, padx=10, pady=5)

        left = tk.Frame(main, bg="#f0f0f0")
        left.pack(side="left", fill="y", padx=(0, 10))

        right = tk.Frame(main, bg="#f0f0f0")
        right.pack(side="left", fill="both", expand=True)

        # --- Input Section ---
        input_frame = ttk.LabelFrame(left, text="Add Process", padding=10)
        input_frame.pack(fill="x", pady=(0, 5))

        tk.Label(input_frame, text="Process ID", bg="#f0f0f0", fg="black").grid(row=0, column=0, sticky="w", pady=2)
        self.pid_entry = tk.Entry(input_frame, width=14, bg="white", fg="black", insertbackground="black")
        self.pid_entry.grid(row=0, column=1, pady=2, padx=(5, 0))
        self.pid_entry.insert(0, "P1")

        tk.Label(input_frame, text="Arrival Time", bg="#f0f0f0", fg="black").grid(row=1, column=0, sticky="w", pady=2)
        self.arrival_entry = tk.Entry(input_frame, width=14, bg="white", fg="black", insertbackground="black")
        self.arrival_entry.grid(row=1, column=1, pady=2, padx=(5, 0))

        tk.Label(input_frame, text="Burst Time", bg="#f0f0f0", fg="black").grid(row=2, column=0, sticky="w", pady=2)
        self.burst_entry = tk.Entry(input_frame, width=14, bg="white", fg="black", insertbackground="black")
        self.burst_entry.grid(row=2, column=1, pady=2, padx=(5, 0))

        ttk.Button(input_frame, text="Add Process", command=self.add_process,
                   style="Blue.TButton").grid(
            row=3, column=0, columnspan=2, sticky="ew", pady=(8, 0))

        # --- Process Table ---
        table_frame = ttk.LabelFrame(left, text="Process List", padding=5)
        table_frame.pack(fill="x", pady=(0, 5))

        cols = ("PID", "Arrival", "Burst")
        self.proc_table = ttk.Treeview(table_frame, columns=cols, show="headings", height=5)
        for c in cols:
            self.proc_table.heading(c, text=c)
            self.proc_table.column(c, width=70, anchor="center")
        self.proc_table.pack(fill="x")

        # --- Algorithm Selection ---
        algo_frame = ttk.LabelFrame(left, text="Select Algorithm", padding=10)
        algo_frame.pack(fill="x", pady=(0, 5))

        self.algo_var = tk.StringVar(value="FCFS")
        for algo in ("FCFS", "SJF", "SJF Preemptive", "Round Robin"):
            tk.Radiobutton(algo_frame, text=algo, variable=self.algo_var, value=algo,
                          bg="#f0f0f0", fg="black", selectcolor="white",
                          activebackground="#f0f0f0", command=self.on_algo_change).pack(anchor="w")

        q_frame = tk.Frame(algo_frame, bg="#f0f0f0")
        q_frame.pack(fill="x", pady=(5, 0))
        tk.Label(q_frame, text="Time Quantum:", bg="#f0f0f0", fg="black").pack(side="left")
        self.quantum_entry = tk.Entry(q_frame, width=5, bg="white", fg="black",
                                       disabledbackground="#d0d0d0", disabledforeground="gray",
                                       insertbackground="black")
        self.quantum_entry.pack(side="left", padx=(5, 0))
        self.quantum_entry.insert(0, "2")
        self.quantum_entry.config(state="disabled")

        # --- Buttons ---
        btn_frame = tk.Frame(left, bg="#f0f0f0")
        btn_frame.pack(fill="x", pady=(0, 5))

        ttk.Button(btn_frame, text="Load Example", command=self.load_example,
                   style="Orange.TButton").pack(side="left", expand=True, fill="x", padx=(0, 3))
        ttk.Button(btn_frame, text="Run Simulation", command=self.run_simulation,
                   style="Green.TButton").pack(side="left", expand=True, fill="x", padx=(0, 3))
        ttk.Button(btn_frame, text="Reset", command=self.reset,
                   style="Red.TButton").pack(side="left", expand=True, fill="x")

        # --- Gantt Chart ---
        gantt_frame = ttk.LabelFrame(right, text="Gantt Chart", padding=5)
        gantt_frame.pack(fill="x", pady=(0, 5))
        self.gantt_canvas = tk.Canvas(gantt_frame, height=80, bg="white")
        self.gantt_canvas.pack(fill="x")

        # --- Results Table ---
        result_frame = ttk.LabelFrame(right, text="Results", padding=5)
        result_frame.pack(fill="both", expand=True)

        res_cols = ("PID", "Arrival", "Burst", "Completion", "Turnaround", "Waiting")
        self.result_table = ttk.Treeview(result_frame, columns=res_cols, show="headings", height=8)
        for c in res_cols:
            self.result_table.heading(c, text=c)
            self.result_table.column(c, width=80, anchor="center")
        self.result_table.pack(fill="both", expand=True)

        self.avg_label = tk.Label(result_frame, text="", font=("Arial", 11, "bold"),
                                   bg="white", fg="black", anchor="w", padx=5)
        self.avg_label.pack(fill="x", pady=(3, 0))

    def on_algo_change(self):
        if self.algo_var.get() == "Round Robin":
            self.quantum_entry.config(state="normal")
        else:
            self.quantum_entry.config(state="disabled")

    def add_process(self):
        pid = self.pid_entry.get().strip()
        if not pid:
            messagebox.showwarning("Error", "Process ID cannot be empty.")
            return
        if any(p.pid == pid for p in self.processes):
            messagebox.showwarning("Error", f"Process '{pid}' already exists.")
            return

        try:
            arrival = int(self.arrival_entry.get().strip())
            burst = int(self.burst_entry.get().strip())
            if arrival < 0 or burst <= 0:
                raise ValueError
        except ValueError:
            messagebox.showwarning("Error", "Arrival Time must be >= 0 and Burst Time must be > 0.")
            return

        proc = Process(pid, arrival, burst)
        self.processes.append(proc)
        self.proc_table.insert("", "end", values=(pid, arrival, burst))

        # auto increment pid
        self.pid_entry.delete(0, "end")
        self.pid_entry.insert(0, f"P{len(self.processes) + 1}")
        self.arrival_entry.delete(0, "end")
        self.burst_entry.delete(0, "end")

    def run_simulation(self):
        if not self.processes:
            messagebox.showwarning("Error", "Please add at least one process.")
            return

        algo = self.algo_var.get()
        try:
            if algo == "FCFS":
                results, gantt = fcfs(self.processes)
            elif algo == "SJF":
                results, gantt = sjf(self.processes)
            elif algo == "SJF Preemptive":
                results, gantt = sjf_preemptive(self.processes)
            else:
                quantum = int(self.quantum_entry.get().strip())
                if quantum <= 0:
                    raise ValueError
                results, gantt = round_robin(self.processes, quantum)
        except ValueError:
            messagebox.showwarning("Error", "Time Quantum must be a positive integer.")
            return

        # assign colors to processes
        pids = sorted(set(p.pid for p in results))
        self.color_map = {pid: COLORS[i % len(COLORS)] for i, pid in enumerate(pids)}

        self.draw_gantt(gantt)
        self.show_results(results)

    def draw_gantt(self, gantt):
        canvas = self.gantt_canvas
        canvas.delete("all")
        if not gantt:
            return

        total_time = gantt[-1].end
        canvas.update_idletasks()
        width = canvas.winfo_width() - 20
        if width < 100:
            width = 600
        y0 = 10
        y1 = 50
        x_off = 10
        scale = width / max(total_time, 1)

        for blk in gantt:
            x0 = x_off + blk.start * scale
            x1 = x_off + blk.end * scale
            color = IDLE_COLOR if blk.pid == "Idle" else self.color_map.get(blk.pid, "#90CAF9")

            canvas.create_rectangle(x0, y0, x1, y1, fill=color, outline="white", width=2)
            mid_x = (x0 + x1) / 2
            canvas.create_text(mid_x, (y0 + y1) / 2, text=blk.pid, font=("Arial", 9, "bold"))
            canvas.create_text(x0, y1 + 12, text=str(blk.start), font=("Arial", 8))
            canvas.create_text(x1, y1 + 12, text=str(blk.end), font=("Arial", 8))

    def show_results(self, results):
        self.result_table.delete(*self.result_table.get_children())
        total_tat = 0
        total_wt = 0

        for p in sorted(results, key=lambda p: p.pid):
            self.result_table.insert("", "end", values=(
                p.pid, p.arrival, p.burst, p.completion, p.turnaround, p.waiting))
            total_tat += p.turnaround
            total_wt += p.waiting

        n = len(results)
        avg_tat = total_tat / n
        avg_wt = total_wt / n
        self.avg_label.config(
            text=f"Avg Turnaround Time: {avg_tat:.2f}   |   Avg Waiting Time: {avg_wt:.2f}")

    def load_example(self):
        algo = self.algo_var.get()
        example = EXAMPLES.get(algo, [])
        # reset first
        self.reset()
        for p in example:
            proc = Process(p.pid, p.arrival, p.burst)
            self.processes.append(proc)
            self.proc_table.insert("", "end", values=(proc.pid, proc.arrival, proc.burst))
        # update pid entry
        self.pid_entry.delete(0, "end")
        self.pid_entry.insert(0, f"P{len(self.processes) + 1}")

    def reset(self):
        self.processes.clear()
        self.color_map.clear()
        self.proc_table.delete(*self.proc_table.get_children())
        self.result_table.delete(*self.result_table.get_children())
        self.gantt_canvas.delete("all")
        self.avg_label.config(text="")
        self.pid_entry.delete(0, "end")
        self.pid_entry.insert(0, "P1")
        self.arrival_entry.delete(0, "end")
        self.burst_entry.delete(0, "end")


# run the app
if __name__ == "__main__":
    app = SchedulerApp()
    app.mainloop()
