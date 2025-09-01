import tkinter as tk
from tkinter import ttk, messagebox, simpledialog
import json, os, hashlib
from datetime import datetime, date

# =================== Config & Estilo ===================
APP_NAME = "Lumin Bank"
DB_FILE = "banco_de_dados.json"
THEME_BG = "#0F1420"
CARD_BG = "#151D2E"
FG = "#EAF0FF"
ACCENT  = "#00FF00"
ACCENT_2 = "#0066FF"
DANGER = "#FF0000"

# =================== Util ===================
def sha(s: str) -> str:
    return hashlib.sha256(s.encode()).hexdigest()

def nowstr() -> str:
    return datetime.now().strftime('%d/%m/%Y %H:%M:%S')

# =================== Banco de Dados ===================
class LuminDB:
    def __init__(self):
        self.data = {"users": {}}
        self._load()
        if not self.data["users"]:
            # Seed com 2 contas exemplo
            self.create_user("lumi", "1234", idade=16, saldo=150.0)
            self.create_user("nova", "abcd", idade=19, saldo=300.0)

    def _load(self):
        if os.path.exists(DB_FILE):
            with open(DB_FILE, 'r', encoding='utf-8') as f:
                self.data = json.load(f)
        else:
            self._save()

    def _save(self):
        with open(DB_FILE, 'w', encoding='utf-8') as f:
            json.dump(self.data, f, indent=2, ensure_ascii=False)

    # ---------- Usúario ----------
    def create_user(self, username: str, password: str, idade: int, saldo: float = 0.0):
        u = username.lower().strip()
        if not u or u in self.data["users"]:
            return False, "Usuário inválido ou já existe."
        if len(password) < 4:
            return False, "Senha deve ter 4+ caracteres."
        if idade < 10 or idade > 100:
            return False, "Idade fora do intervalo (10-100)."
        self.data["users"][u] = {
            "hash": sha(password),
            "idade": idade,
            "saldo": float(saldo),
            "dividas": 0.0,
            "daily_limit": 100.0 if idade < 18 else 999999.0,
            "spent_today": 0.0,
            "spent_date": date.today().isoformat(),
            "goals": [],  
            "badges": [],
            "categories": {},  # categoria -> total gasto
            "history": []
        }
        self._log(u, f"Conta criada | saldo inicial R$ {saldo:.2f} | idade {idade}")
        self._badge_check(u)
        self._save()
        return True, "Conta criada com sucesso!"

    def auth(self, username: str, password: str):
        u = username.lower().strip()
        user = self.data["users"].get(u)
        if not user:
            return False
        return user["hash"] == sha(password)

    def _ensure_day(self, u):
        user = self.data["users"][u]
        today = date.today().isoformat()
        if user["spent_date"] != today:
            user["spent_date"] = today
            user["spent_today"] = 0.0

    def _log(self, u, msg):
        self.data["users"][u]["history"].append(f"[{nowstr()}] {msg}")

    def _badge_add(self, u, badge):
        b = self.data["users"][u]["badges"]
        if badge not in b:
            b.append(badge)
            self._log(u, f"Conquista desbloqueada: {badge}")

    def _badge_check(self, u):
        user = self.data["users"][u]
        if user["saldo"] >= 100:
            self._badge_add(u, "Primeiros 100")
        if len(user["history"]) >= 10:
            self._badge_add(u, "Ativo x10")
        # meta concluída vira estatisticas na função de metas

    # ---------- verdinha ----------
    def deposit(self, u, valor: float):
        if valor <= 0:
            return False, "Valor inválido."
        self.data["users"][u]["saldo"] += valor
        self._log(u, f"Depósito R$+ {valor:.2f}")
        self._badge_check(u)
        self._save()
        return True, "Depósito realizado."

    def spend(self, u, valor: float, categoria: str = "geral"):
        if valor <= 0:
            return False, "Valor inválido."
        self._ensure_day(u)
        user = self.data["users"][u]
        if valor > user["saldo"]:
            return False, "Saldo insuficiente."
        # Limite diário para menores de idade
        if user["idade"] < 18 and (user["spent_today"] + valor) > user["daily_limit"]:
            return False, f"Limite diário de R$ {user['daily_limit']:.2f} excedido."
        user["saldo"] -= valor
        user["spent_today"] += valor
        user["categories"][categoria] = user["categories"].get(categoria, 0.0) + valor
        self._log(u, f"Gasto -R$ {valor:.2f} | cat: {categoria}")
        self._badge_check(u)
        self._save()
        return True, "Gasto registrado."

    def transfer(self, origem, destino, valor: float):
        origem = origem.lower(); destino = destino.lower()
        if origem == destino:
            return False, "Não transfira para você mesmo."
        if destino not in self.data["users"]:
            return False, "Destinatário não existe."
        if valor <= 0:
            return False, "Valor inválido."
        if valor > self.data["users"][origem]["saldo"]:
            return False, "Saldo insuficiente."
        self.data["users"][origem]["saldo"] -= valor
        self.data["users"][destino]["saldo"] += valor
        self._log(origem, f"Transferiu -R$ {valor:.2f} para @{destino}")
        self._log(destino, f"Recebeu +R$ {valor:.2f} de @{origem}")
        self._badge_check(origem); self._badge_check(destino)
        self._save()
        return True, "Transferência realizada."

    def pay_debt(self, u, valor: float):
        user = self.data["users"][u]
        if valor <= 0:
            return False, "Valor inválido."
        if valor > user["saldo"]:
            return False, "Saldo insuficiente."
        if user["dividas"] <= 0:
            return False, "Você não possui dívidas."
        valor = min(valor, user["dividas"])  # pagamento parcial permitido
        user["saldo"] -= valor
        user["dividas"] -= valor
        self._log(u, f"Pagamento de dívida R$ {valor:.2f}")
        self._save()
        return True, "Dívida paga (parcial/total)."

    def loan(self, u, valor: float, juros=0.10):
        if valor <= 0:
            return False, "Valor inválido."
        j = valor * juros
        total = valor + j
        self.data["users"][u]["saldo"] += valor
        self.data["users"][u]["dividas"] += total
        self._log(u, f"Empréstimo +R$ {valor:.2f} | juros R$ {j:.2f} | dívida +R$ {total:.2f}")
        self._save()
        return True, f"Empréstimo concedido. Você deve R$ {total:.2f}."

    # ---------- Metas ----------
    def goal_add(self, u, name: str, target: float):
        if not name or target <= 0:
            return False, "Dados da meta inválidos."
        g = {"name": name, "target": float(target), "saved": 0.0, "done": False}
        self.data["users"][u]["goals"].append(g)
        self._log(u, f"Meta criada: {name} (R$ {target:.2f})")
        self._save()
        return True, "Meta adicionada."

    def goal_deposit(self, u, idx: int, valor: float):
        goals = self.data["users"][u]["goals"]
        if idx < 0 or idx >= len(goals):
            return False, "Meta não encontrada."
        if valor <= 0:
            return False, "Valor inválido."
        user = self.data["users"][u]
        if valor > user["saldo"]:
            return False, "Saldo insuficiente."
        g = goals[idx]
        user["saldo"] -= valor
        g["saved"] += valor
        if g["saved"] >= g["target"] and not g["done"]:
            g["done"] = True
            self._log(u, f"Meta concluída: {g['name']}")
            self._badge_add(u, "Meta Concluída")
        else:
            self._log(u, f"Depósito na meta '{g['name']}' R$ {valor:.2f}")
        self._badge_check(u)
        self._save()
        return True, "Valor enviado para a meta."

    # ---------- Relatório ----------
    def user_summary(self, u):
        user = self.data["users"][u]
        disp = user["saldo"] - user["dividas"]
        return (
            f"Saldo: R$ {user['saldo']:.2f}\n"
            f"Dívidas: R$ {user['dividas']:.2f}\n"
            f"Disponível: R$ {disp:.2f}\n"
            f"Idade: {user['idade']} | Limite diário: R$ {user['daily_limit']:.2f}\n"
            f"Metas: {len(user['goals'])} | Conquistas: {', '.join(user['badges']) if user['badges'] else '—'}\n"
        )

    def export_history(self, u):
        fn = f"historico_{u}.txt"
        with open(fn, 'w', encoding='utf-8') as f:
            f.write("\n".join(self.data["users"][u]["history"]))
        return fn

# =================== UI ===================
class LuminApp:
    def __init__(self, root):
        self.db = LuminDB()
        self.user = None
        self.root = root
        root.title(APP_NAME)
        root.geometry('980x640')
        root.configure(bg=THEME_BG)
        root.resizable(False, False)

        # cabeçalho
        self.header = tk.Frame(root, bg=THEME_BG)
        self.header.pack(fill='x', pady=10)
        tk.Label(self.header, text=APP_NAME, fg=FG, bg=THEME_BG,
                 font=('Segoe UI', 28, 'bold')).pack(side='left', padx=20)
        self.status_lbl = tk.Label(self.header, text='Faça login', fg=ACCENT, bg=THEME_BG, font=('Segoe UI', 12))
        self.status_lbl.pack(side='right', padx=20)

        # meio
        self.container = tk.Frame(root, bg=THEME_BG)
        self.container.pack(expand=True, fill='both', padx=16, pady=6)

        self.show_login()

    # --------- Funções UI ---------
    def card(self, parent):
        f = tk.Frame(parent, bg=CARD_BG, bd=0, highlightthickness=0)
        f.pack(fill='both', expand=True, padx=8, pady=8)
        return f

    def btn(self, parent, text, cmd, color=ACCENT):
        b = tk.Button(parent, text=text, command=cmd, bg=color, fg='#0b1220',
                      font=('Segoe UI', 12, 'bold'), bd=0, padx=14, pady=10, activebackground=color)
        return b

    # --------- Telas ---------
    def show_login(self):
        for w in self.container.winfo_children(): w.destroy()
        card = self.card(self.container)

        tk.Label(card, text='Bem-vindo ao Lumin Bank', bg=CARD_BG, fg=FG, font=('Segoe UI', 22, 'bold')).pack(pady=(24,8))
        tk.Label(card, text='Banco digital pensado para jovens', bg=CARD_BG, fg=FG, font=('Segoe UI', 12)).pack(pady=(0,16))

        frm = tk.Frame(card, bg=CARD_BG)
        frm.pack(pady=8)
        tk.Label(frm, text='Usuário', bg=CARD_BG, fg=FG, font=('Segoe UI', 12)).grid(row=0, column=0, sticky='w', padx=6, pady=6)
        self.ent_user = tk.Entry(frm, width=24, font=('Segoe UI', 14))
        self.ent_user.grid(row=0, column=1, padx=6, pady=6)

        tk.Label(frm, text='Senha', bg=CARD_BG, fg=FG, font=('Segoe UI', 12)).grid(row=1, column=0, sticky='w', padx=6, pady=6)
        self.ent_pass = tk.Entry(frm, width=24, font=('Segoe UI', 14), show='*')
        self.ent_pass.grid(row=1, column=1, padx=6, pady=6)

        btns = tk.Frame(card, bg=CARD_BG)
        btns.pack(pady=10)
        self.btn(btns, 'Entrar', self.do_login).grid(row=0, column=0, padx=6)
        self.btn(btns, 'Criar conta', self.show_signup, color=ACCENT_2).grid(row=0, column=1, padx=6)

    def show_signup(self):
        for w in self.container.winfo_children(): w.destroy()
        card = self.card(self.container)
        tk.Label(card, text='Criar conta', bg=CARD_BG, fg=FG, font=('Segoe UI', 22, 'bold')).pack(pady=(24,8))
        frm = tk.Frame(card, bg=CARD_BG)
        frm.pack(pady=8)
        tk.Label(frm, text='Usuário', bg=CARD_BG, fg=FG).grid(row=0, column=0, sticky='w', padx=6, pady=6)
        self.su_user = tk.Entry(frm, width=24, font=('Segoe UI', 14))
        self.su_user.grid(row=0, column=1, padx=6, pady=6)
        tk.Label(frm, text='Senha', bg=CARD_BG, fg=FG).grid(row=1, column=0, sticky='w', padx=6, pady=6)
        self.su_pass = tk.Entry(frm, width=24, font=('Segoe UI', 14), show='*')
        self.su_pass.grid(row=1, column=1, padx=6, pady=6)
        tk.Label(frm, text='Idade', bg=CARD_BG, fg=FG).grid(row=2, column=0, sticky='w', padx=6, pady=6)
        self.su_age = tk.Entry(frm, width=8, font=('Segoe UI', 14))
        self.su_age.grid(row=2, column=1, sticky='w', padx=6, pady=6)

        tk.Label(frm, text='Saldo inicial (opcional)', bg=CARD_BG, fg=FG).grid(row=3, column=0, sticky='w', padx=6, pady=6)
        self.su_balance = tk.Entry(frm, width=12, font=('Segoe UI', 14))
        self.su_balance.grid(row=3, column=1, sticky='w', padx=6, pady=6)

        self.btn(card, 'Salvar conta', self.do_signup).pack(pady=12)
        self.btn(card, 'Voltar', self.show_login, color=DANGER).pack(pady=(0,12))

    def show_main(self):
        for w in self.container.winfo_children(): w.destroy()
        nb = ttk.Notebook(self.container)
        nb.pack(expand=True, fill='both')
        style = ttk.Style(); style.theme_use('default')
        style.configure('TNotebook', background=THEME_BG)
        style.configure('TNotebook.Tab', padding=[14, 8])

        self.tab_dash = tk.Frame(nb, bg=THEME_BG)
        self.tab_wallet = tk.Frame(nb, bg=THEME_BG)
        self.tab_goals = tk.Frame(nb, bg=THEME_BG)
        self.tab_hist = tk.Frame(nb, bg=THEME_BG)
        self.tab_settings = tk.Frame(nb, bg=THEME_BG)

        nb.add(self.tab_dash, text='Dashboard')
        nb.add(self.tab_wallet, text='Carteira')
        nb.add(self.tab_goals, text='Metas')
        nb.add(self.tab_hist, text='Histórico')
        nb.add(self.tab_settings, text='Configurações')

        self.build_dashboard()
        self.build_wallet()
        self.build_goals()
        self.build_history()
        self.build_settings()

    # --------- Sumario ---------
    def build_dashboard(self):
        for w in self.tab_dash.winfo_children(): w.destroy()
        card = self.card(self.tab_dash)
        u = self.user
        summary = self.db.user_summary(u)
        tk.Label(card, text=f"Olá, @{u}!", bg=CARD_BG, fg=ACCENT, font=('Segoe UI', 20, 'bold')).pack(anchor='w', padx=16, pady=(16,4))
        tk.Label(card, text=summary, bg=CARD_BG, fg=FG, font=('Consolas', 12), justify='left').pack(anchor='w', padx=16, pady=(0,8))

        # Ações rápidas
        actions = tk.Frame(card, bg=CARD_BG)
        actions.pack(padx=12, pady=12, anchor='w')
        self.btn(actions, 'Depositar +50', lambda: self.quick_action(self.db.deposit, 50)).grid(row=0, column=0, padx=6, pady=6)
        self.btn(actions, 'Gastar 25 (lanches)', lambda: self.quick_action(lambda u,v: self.db.spend(u,v,'lanches'), 25), color=ACCENT_2).grid(row=0, column=1, padx=6, pady=6)
        self.btn(actions, 'Pagar dívida 20', lambda: self.quick_action(self.db.pay_debt, 20), color='#FFD166').grid(row=0, column=2, padx=6, pady=6)

    def quick_action(self, fn, valor):
        ok, msg = fn(self.user, valor)
        messagebox.showinfo('Lumin', msg) if ok else messagebox.showerror('Lumin', msg)
        self.build_dashboard(); self.build_wallet(); self.build_history()

    def build_wallet(self):
        for w in self.tab_wallet.winfo_children(): w.destroy()
        card = self.card(self.tab_wallet)
        tk.Label(card, text='Carteira', bg=CARD_BG, fg=FG, font=('Segoe UI', 18, 'bold')).pack(anchor='w', padx=16, pady=(16,8))

        row = tk.Frame(card, bg=CARD_BG); row.pack(anchor='w', padx=16, pady=4)
        tk.Label(row, text='Valor:', bg=CARD_BG, fg=FG).grid(row=0, column=0, padx=6, pady=6)
        self.amount = tk.Entry(row, width=12, font=('Segoe UI', 12)); self.amount.grid(row=0, column=1, padx=6)
        tk.Label(row, text='Categoria (gasto):', bg=CARD_BG, fg=FG).grid(row=0, column=2, padx=6)
        self.cat = tk.Entry(row, width=14, font=('Segoe UI', 12)); self.cat.grid(row=0, column=3, padx=6)

        actions = tk.Frame(card, bg=CARD_BG); actions.pack(anchor='w', padx=16, pady=8)
        self.btn(actions, 'Depositar', self.ui_deposit).grid(row=0, column=0, padx=6, pady=6)
        self.btn(actions, 'Gastar', self.ui_spend, color=ACCENT_2).grid(row=0, column=1, padx=6, pady=6)
        self.btn(actions, 'Pagar dívida', self.ui_paydebt, color="#C7CA13").grid(row=0, column=2, padx=6, pady=6)
        self.btn(actions, 'Empréstimo', self.ui_loan, color="#04B801").grid(row=0, column=3, padx=6, pady=6)

        # Transferência
        tcard = self.card(self.tab_wallet)
        tk.Label(tcard, text='Transferência', bg=CARD_BG, fg=FG, font=('Segoe UI', 16, 'bold')).pack(anchor='w', padx=16, pady=(16,8))
        row2 = tk.Frame(tcard, bg=CARD_BG); row2.pack(anchor='w', padx=16, pady=4)
        tk.Label(row2, text='Para (usuário):', bg=CARD_BG, fg=FG).grid(row=0, column=0, padx=6, pady=6)
        self.dest = tk.Entry(row2, width=18, font=('Segoe UI', 12)); self.dest.grid(row=0, column=1, padx=6)
        tk.Label(row2, text='Valor:', bg=CARD_BG, fg=FG).grid(row=0, column=2, padx=6)
        self.tval = tk.Entry(row2, width=12, font=('Segoe UI', 12)); self.tval.grid(row=0, column=3, padx=6)
        self.btn(row2, 'Enviar', self.ui_transfer).grid(row=0, column=4, padx=8)

    def build_goals(self):
        for w in self.tab_goals.winfo_children(): w.destroy()
        card = self.card(self.tab_goals)
        tk.Label(card, text='Metas de economia', bg=CARD_BG, fg=FG, font=('Segoe UI', 18, 'bold')).pack(anchor='w', padx=16, pady=(16,8))

        # lista de metas
        self.goals_box = tk.Listbox(card, height=8, font=('Segoe UI', 12))
        self.goals_box.pack(fill='x', padx=16)
        self.refresh_goals_list()

        controls = tk.Frame(card, bg=CARD_BG); controls.pack(anchor='w', padx=16, pady=8)
        self.btn(controls, 'Nova meta', self.ui_goal_add).grid(row=0, column=0, padx=6)
        self.btn(controls, 'Depositar na meta', self.ui_goal_deposit, color=ACCENT_2).grid(row=0, column=1, padx=6)

    def build_history(self):
        for w in self.tab_hist.winfo_children(): w.destroy()
        card = self.card(self.tab_hist)
        tk.Label(card, text='Histórico', bg=CARD_BG, fg=FG, font=('Segoe UI', 18, 'bold')).pack(anchor='w', padx=16, pady=(16,8))
        hist = self.db.data["users"][self.user]["history"]
        text = tk.Text(card, height=18, bg='#0c1322', fg=FG, insertbackground=FG)
        text.pack(fill='both', padx=16, pady=8, expand=True)
        text.insert('1.0', "\n".join(hist) if hist else "Sem operações ainda.")
        text.config(state='disabled')

        self.btn(card, 'Exportar histórico (.txt)', self.ui_export).pack(pady=8, anchor='w', padx=16)

    def build_settings(self):
        for w in self.tab_settings.winfo_children(): w.destroy()
        card = self.card(self.tab_settings)
        tk.Label(card, text='Configurações', bg=CARD_BG, fg=FG, font=('Segoe UI', 18, 'bold')).pack(anchor='w', padx=16, pady=(16,8))

        user = self.db.data["users"][self.user]
        row = tk.Frame(card, bg=CARD_BG); row.pack(anchor='w', padx=16, pady=6)
        tk.Label(row, text='Limite diário (menores de 18):', bg=CARD_BG, fg=FG).grid(row=0, column=0, padx=6)
        self.limit_entry = tk.Entry(row, width=10); self.limit_entry.grid(row=0, column=1, padx=6)
        self.limit_entry.insert(0, f"{user['daily_limit']:.2f}")
        self.btn(row, 'Salvar limite', self.ui_save_limit).grid(row=0, column=2, padx=8)

        self.btn(card, 'Sair da conta', self.logout, color=DANGER).pack(padx=16, pady=16, anchor='w')

    # --------- Ações---------
    def do_login(self):
        u = self.ent_user.get().strip().lower()
        p = self.ent_pass.get()
        if not self.db.auth(u, p):
            messagebox.showerror('Lumin', 'Usuário ou senha inválidos.')
            return
        self.user = u
        self.status_lbl.config(text=f"Logado: @{u}")
        self.show_main()

    def do_signup(self):
        u = self.su_user.get().strip().lower()
        p = self.su_pass.get()
        try:
            idade = int(self.su_age.get().strip())
        except:
            messagebox.showerror('Lumin', 'Informe uma idade válida.'); return
        saldo = 0.0
        if self.su_balance.get().strip():
            try:
                saldo = float(self.su_balance.get().replace(',', '.'))
            except:
                messagebox.showerror('Lumin', 'Saldo inicial inválido.'); return
        ok, msg = self.db.create_user(u, p, idade, saldo)
        (messagebox.showinfo if ok else messagebox.showerror)('Lumin', msg)
        if ok:
            self.show_login()

    def ui_deposit(self):
        v = self._read_amount(self.amount)
        if v is None: return
        ok, msg = self.db.deposit(self.user, v)
        (messagebox.showinfo if ok else messagebox.showerror)('Lumin', msg)
        self.build_dashboard(); self.build_wallet(); self.build_history()

    def ui_spend(self):
        v = self._read_amount(self.amount)
        if v is None: return
        cat = self.cat.get().strip() or 'geral'
        ok, msg = self.db.spend(self.user, v, cat)
        (messagebox.showinfo if ok else messagebox.showerror)('Lumin', msg)
        self.build_dashboard(); self.build_wallet(); self.build_history()

    def ui_paydebt(self):
        v = self._read_amount(self.amount)
        if v is None: return
        ok, msg = self.db.pay_debt(self.user, v)
        (messagebox.showinfo if ok else messagebox.showerror)('Lumin', msg)
        self.build_dashboard(); self.build_wallet(); self.build_history()

    def ui_loan(self):
        v = self._read_amount(self.amount)
        if v is None: return
        ok, msg = self.db.loan(self.user, v)
        (messagebox.showinfo if ok else messagebox.showerror)('Lumin', msg)
        self.build_dashboard(); self.build_wallet(); self.build_history()

    def ui_transfer(self):
        d = self.dest.get().strip().lower()
        try:
            v = float(self.tval.get().replace(',', '.'))
        except:
            messagebox.showerror('Lumin', 'Valor inválido.'); return
        ok, msg = self.db.transfer(self.user, d, v)
        (messagebox.showinfo if ok else messagebox.showerror)('Lumin', msg)
        self.build_dashboard(); self.build_wallet(); self.build_history()

    def ui_goal_add(self):
        name = simpledialog.askstring('Lumin', 'Nome da meta:')
        if not name: return
        try:
            target = float(simpledialog.askstring('Lumin', 'Valor alvo (R$):').replace(',', '.'))
        except:
            messagebox.showerror('Lumin', 'Valor inválido.'); return
        ok, msg = self.db.goal_add(self.user, name, target)
        (messagebox.showinfo if ok else messagebox.showerror)('Lumin', msg)
        self.refresh_goals_list(); self.build_dashboard()

    def ui_goal_deposit(self):
        idx = self.goals_box.curselection()
        if not idx:
            messagebox.showerror('Lumin', 'Selecione uma meta.'); return
        try:
            v = float(simpledialog.askstring('Lumin', 'Valor para depositar:').replace(',', '.'))
        except:
            messagebox.showerror('Lumin', 'Valor inválido.'); return
        ok, msg = self.db.goal_deposit(self.user, idx[0], v)
        (messagebox.showinfo if ok else messagebox.showerror)('Lumin', msg)
        self.refresh_goals_list(); self.build_dashboard(); self.build_history()

    def refresh_goals_list(self):
        self.goals_box.delete(0, 'end')
        goals = self.db.data["users"][self.user]["goals"]
        for i, g in enumerate(goals):
            pct = min(100, int((g['saved'] / g['target']) * 100)) if g['target'] > 0 else 0
            done = '✅' if g['done'] else f"{pct}%"
            self.goals_box.insert('end', f"{i+1}. {g['name']} | R$ {g['saved']:.2f} / {g['target']:.2f} ({done})")

    def ui_export(self):
        fn = self.db.export_history(self.user)
        messagebox.showinfo('Lumin', f'Histórico exportado como {fn}.')

    def ui_save_limit(self):
        try:
            v = float(self.limit_entry.get().replace(',', '.'))
        except:
            messagebox.showerror('Lumin', 'Valor inválido.'); return
        user = self.db.data["users"][self.user]
        user['daily_limit'] = max(10.0, v)
        self.db._save()
        messagebox.showinfo('Lumin', 'Limite salvo!')

    def logout(self):
        self.user = None
        self.status_lbl.config(text='Faça login')
        self.show_login()

    # auxiliadores
    def _read_amount(self, entry: tk.Entry):
        try:
            return float(entry.get().replace(',', '.'))
        except:
            messagebox.showerror('Lumin', 'Informe um valor numérico.'); return None
            
if __name__ == '__main__':
    root = tk.Tk()
    app = LuminApp(root)
    root.state('zoomed') 
    root.bind("<Escape>", lambda e: root.attributes("-fullscreen", False))  
    root.mainloop()
