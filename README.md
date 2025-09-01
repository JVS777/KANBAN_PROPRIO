# KANBAN_PROPRIO

import sys
import sqlite3
from PyQt5.QtWidgets import (
    QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
    QLabel, QPushButton, QListWidget, QListWidgetItem,
    QDialog, QFormLayout, QDateEdit, QDialogButtonBox, QMenu, QLineEdit
)
from PyQt5.QtCore import Qt, QDate
from PyQt5.QtGui import QFont


DB_FILE = "kanban.db"
DEFAULT_COLUMNS = ["A Fazer", "Em Andamento", "Concluído"]


# ---------- BANCO DE DADOS ----------
def init_db():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("""
        CREATE TABLE IF NOT EXISTS columns (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL
        )
    """)
    c.execute("""
        CREATE TABLE IF NOT EXISTS tasks (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            column_id INTEGER,
            title TEXT NOT NULL,
            start_date TEXT,
            end_date TEXT,
            category TEXT,
            FOREIGN KEY (column_id) REFERENCES columns (id)
        )
    """)
    c.execute("SELECT COUNT(*) FROM columns")
    if c.fetchone()[0] == 0:
        for name in DEFAULT_COLUMNS:
            c.execute("INSERT INTO columns (name) VALUES (?)", (name,))
    conn.commit()
    conn.close()




# ---------- FORMULÁRIO DE TAREFA ----------
class TaskDialog(QDialog):
    def __init__(self, task=None):
        super().__init__()
        self.setWindowTitle("Editar Tarefa")
        self.task = task
        layout = QFormLayout(self)


        self.title_input = QLineEdit(self)
        self.start_date = QDateEdit(self)
        self.start_date.setCalendarPopup(True)
        self.end_date = QDateEdit(self)
        self.end_date.setCalendarPopup(True)
        self.category = QLineEdit(self)
        self.category.setPlaceholderText("Digite a categoria...")


        layout.addRow("Título:", self.title_input)
        layout.addRow("Data Início:", self.start_date)
        layout.addRow("Data Fim:", self.end_date)
        layout.addRow("Categoria:", self.category)


        buttons = QDialogButtonBox(QDialogButtonBox.Ok | QDialogButtonBox.Cancel, self)
        buttons.accepted.connect(self.accept)
        buttons.rejected.connect(self.reject)
        layout.addWidget(buttons)


        if task:
            self.title_input.setText(task[2])
            if task[3]:
                self.start_date.setDate(QDate.fromString(task[3], "yyyy-MM-dd"))
            if task[4]:
                self.end_date.setDate(QDate.fromString(task[4], "yyyy-MM-dd"))
            if task[5]:
                self.category.setText(task[5])


    def get_data(self):
        return {
            "title": self.title_input.text(),
            "start_date": self.start_date.date().toString("yyyy-MM-dd"),
            "end_date": self.end_date.date().toString("yyyy-MM-dd"),
            "category": self.category.text()
        }




# ---------- COLUNA ----------
class KanbanColumn(QWidget):
    def __init__(self, column_id, name, parent):
        super().__init__()
        self.column_id = column_id
        self.parent = parent
        self.setFixedWidth(260)
        self.layout = QVBoxLayout(self)
        self.layout.setSpacing(10)
        self.setAcceptDrops(True)


        # Título editável inline
        self.title_label = QLabel(name)
        self.title_label.setAlignment(Qt.AlignCenter)
        self.title_label.setStyleSheet(
            "font-weight:bold; font-size:16px; color:#ffffff; background-color:#42a9e0; padding:6px; border-radius:10px;"
        )
        self.title_label.mouseDoubleClickEvent = self.edit_title_inline
        self.layout.addWidget(self.title_label)


        # Lista de tarefas
        self.task_list = QListWidget()
        self.task_list.setStyleSheet("""
            QListWidget {
                background-color: #eaf6fe;
                border-radius: 12px;
                padding:8px;
            }
            QListWidget::item {
                padding:12px;
                margin:6px 0;
                border-radius:10px;
                background-color:#42a9e0;
                color:#ffffff;
                font-weight:bold;
            }
            QListWidget::item:selected {
                background-color:#302e69;
            }
            QListWidget::item:hover {
                background-color:#6c5ce7;
            }
        """)
        self.task_list.setContextMenuPolicy(Qt.CustomContextMenu)
        self.task_list.customContextMenuRequested.connect(self.show_task_menu)
        self.task_list.itemDoubleClicked.connect(self.edit_task)
        self.task_list.setDragEnabled(True)
        self.task_list.setAcceptDrops(True)
        self.task_list.setDropIndicatorShown(True)
        self.task_list.setDefaultDropAction(Qt.MoveAction)
        self.layout.addWidget(self.task_list)


        self.load_tasks()


        # Campo inline para adicionar tarefa direto
        self.input_task = QLineEdit()
        self.input_task.setPlaceholderText("+ Adicionar tarefa...")
        self.input_task.setStyleSheet("""
            QLineEdit {
                background-color:#ef7d06;
                color:#ffffff;
                border-radius:10px;
                padding:6px;
                font-weight:bold;
            }
            QLineEdit:focus {
                background-color:#e67e22;
            }
        """)
        self.input_task.returnPressed.connect(self.add_task_inline_from_field)
        self.layout.addWidget(self.input_task)


        self.setLayout(self.layout)


    # ---------- Funções ----------
    def edit_title_inline(self, event):
        self.title_label.setTextInteractionFlags(Qt.TextEditorInteraction)
        self.title_label.setFocus()
        self.title_label.editingFinished = self.finish_edit_title


    def finish_edit_title(self):
        text = self.title_label.text().strip()
        if text:
            conn = sqlite3.connect(DB_FILE)
            c = conn.cursor()
            c.execute("UPDATE columns SET name=? WHERE id=?", (text, self.column_id))
            conn.commit()
            conn.close()
        self.title_label.setTextInteractionFlags(Qt.NoTextInteraction)


    def load_tasks(self):
        self.task_list.clear()
        conn = sqlite3.connect(DB_FILE)
        c = conn.cursor()
        c.execute("SELECT * FROM tasks WHERE column_id=?", (self.column_id,))
        for row in c.fetchall():
            item = QListWidgetItem(row[2])
            item.setData(Qt.UserRole, row)
            self.task_list.addItem(item)
        conn.close()


    def add_task_inline_from_field(self):
        text = self.input_task.text().strip()
        if text:
            self.add_task_inline(text)
            self.input_task.clear()


    def add_task_inline(self, text):
        conn = sqlite3.connect(DB_FILE)
        c = conn.cursor()
        c.execute("INSERT INTO tasks (column_id, title) VALUES (?, ?)", (self.column_id, text))
        task_id = c.lastrowid
        conn.commit()
        conn.close()
        item = QListWidgetItem(text)
        item.setData(Qt.UserRole, (task_id, self.column_id, text, None, None, None))
        self.task_list.addItem(item)


    def edit_task(self, item):
        task = item.data(Qt.UserRole)
        dlg = TaskDialog(task)
        if dlg.exec_():
            data = dlg.get_data()
            conn = sqlite3.connect(DB_FILE)
            c = conn.cursor()
            c.execute("""
                UPDATE tasks SET title=?, start_date=?, end_date=?, category=?
                WHERE id=?
            """, (data["title"], data["start_date"], data["end_date"], data["category"], task[0]))
            conn.commit()
            conn.close()
            item.setText(data["title"])
            item.setData(Qt.UserRole, (task[0], self.column_id, data["title"], data["start_date"], data["end_date"], data["category"]))


    def show_task_menu(self, pos):
        item = self.task_list.itemAt(pos)
        if item:
            menu = QMenu()
            delete_action = menu.addAction("Excluir Tarefa")
            action = menu.exec_(self.task_list.mapToGlobal(pos))
            if action == delete_action:
                task = item.data(Qt.UserRole)
                conn = sqlite3.connect(DB_FILE)
                c = conn.cursor()
                c.execute("DELETE FROM tasks WHERE id=?", (task[0],))
                conn.commit()
                conn.close()
                self.task_list.takeItem(self.task_list.row(item))


    # ---------- Drag&Drop corrigido ----------
    def dragEnterEvent(self, event):
        if event.mimeData().hasFormat("application/x-qabstractitemmodeldatalist"):
            event.accept()
        else:
            event.ignore()


    def dropEvent(self, event):
        source = event.source()
        if source == self.task_list:
            event.ignore()
            return


        for item in source.selectedItems():
            task = item.data(Qt.UserRole)
            # Atualiza no DB
            conn = sqlite3.connect(DB_FILE)
            c = conn.cursor()
            c.execute("UPDATE tasks SET column_id=? WHERE id=?", (self.column_id, task[0]))
            conn.commit()
            conn.close()
            # Remove do source e adiciona ao destino
            row = source.row(item)
            source.takeItem(row)
            self.task_list.addItem(item)
        event.accept()


    def contextMenuEvent(self, event):
        menu = QMenu(self)
        del_action = menu.addAction("Excluir Coluna")
        action = menu.exec_(event.globalPos())
        if action == del_action:
            self.setParent(None)
            self.parent.columns_widgets.pop(self.column_id, None)
            conn = sqlite3.connect(DB_FILE)
            c = conn.cursor()
            c.execute("DELETE FROM columns WHERE id=?", (self.column_id,))
            c.execute("DELETE FROM tasks WHERE column_id=?", (self.column_id,))
            conn.commit()
            conn.close()




# ---------- JANELA PRINCIPAL ----------
class KanbanBoard(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Kanban - Drag & Tasks")
        self.resize(1100, 600)
        self.central = QWidget()
        self.setCentralWidget(self.central)
        self.central.setStyleSheet("background-color:#302e69;")
        self.layout = QHBoxLayout(self.central)
        self.layout.setSpacing(15)
        self.columns_widgets = {}
        self.load_columns()


        # Botão minimalista adicionar coluna
        self.btn_add_col = QPushButton("+")
        self.btn_add_col.setFixedSize(50, 50)
        self.btn_add_col.setFont(QFont("Arial", 22, QFont.Bold))
        self.btn_add_col.setStyleSheet("""
            QPushButton {
                background-color:#ef7d06;
                color:#ffffff;
                border-radius:25px;
                border:none;
            }
            QPushButton:hover {
                background-color:#e67e22;
            }
        """)
        self.layout.addWidget(self.btn_add_col)
        self.btn_add_col.clicked.connect(self.add_column_inline)


    def load_columns(self):
        conn = sqlite3.connect(DB_FILE)
        c = conn.cursor()
        c.execute("SELECT * FROM columns")
        for col in c.fetchall():
            self.add_column_widget(col[0], col[1])
        conn.close()


    def add_column_widget(self, col_id, name):
        column_widget = KanbanColumn(col_id, name, self)
        self.layout.insertWidget(self.layout.count() - 1, column_widget)
        self.columns_widgets[col_id] = column_widget


    def add_column_inline(self):
        conn = sqlite3.connect(DB_FILE)
        c = conn.cursor()
        c.execute("INSERT INTO columns (name) VALUES ('Nova Coluna')")
        col_id = c.lastrowid
        conn.commit()
        conn.close()
        self.add_column_widget(col_id, "Nova Coluna")




# ---------- EXECUÇÃO ----------
if __name__ == "__main__":
    init_db()
    app = QApplication(sys.argv)
    board = KanbanBoard()
    board.show()
    sys.exit(app.exec_())
