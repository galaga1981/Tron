---
created: 2024-12-19T13:44:52-06:00
modified: 2024-12-19T13:45:17-06:00
---

# BairdSiteManager2

from textual.app import App, ComposeResult
from textual.containers import Container
from textual.widgets import Button, Header, Footer, Input, Static, Log
import sqlite3
from datetime import datetime


class SiteManagerApp(App):
    """A Textual app to manage site information in the wireless maintenance database."""

    def __init__(self):
        super().__init__()
        self.conn = sqlite3.connect("wireless_maintenance.db")
        self.cursor = self.conn.cursor()
        self.create_table()

    def create_table(self):
        self.cursor.execute("""
        CREATE TABLE IF NOT EXISTS Sites (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            moniker TEXT UNIQUE NOT NULL,
            region TEXT NOT NULL,
            router_ip TEXT NOT NULL,
            switch_ip TEXT NOT NULL,
            data_vlan_id INTEGER NOT NULL,
            wireless_vlan_id INTEGER NOT NULL,
            guest_vlan_id INTEGER NOT NULL,
            created_at DATETIME NOT NULL,
            updated_at DATETIME NOT NULL
        )
        """)
        self.conn.commit()

    def compose(self) -> ComposeResult:
        yield Header(show_clock=True)
        yield Footer()
        yield Container(
            Static("Site Management System", id="title"),
            Input(placeholder="Enter Command (add, view, update, delete)", id="command_input"),
            Static(
                "Examples:\n"
                "Add: add ABC East 192.168.1.1 192.168.1.2 10 20 30\n"
                "View: view\n"
                "Update: update ABC region Midwest\n"
                "Delete: delete ABC\n",
                id="instructions"
            ),
            Button("Submit", id="submit_button"),
            Button("Quit", id="quit_button"),
            Log(id="output_log"),
        )

    async def on_button_pressed(self, event: Button.Pressed):
        log = self.query_one("#output_log", Log)
        input_box = self.query_one("#command_input", Input)

        if event.button.id == "submit_button":
            command = input_box.value.strip().lower()
            input_box.value = ""  # Clear the input box after the command is processed
            if command.startswith("add"):
                await self.add_site(command, log)
            elif command.startswith("view"):
                await self.view_sites(command, log)
            elif command.startswith("update"):
                await self.update_site(command, log)
            elif command.startswith("delete"):
                await self.delete_site(command, log)
            else:
                log.write("Error: Invalid command. Use add, view, update, or delete. Please try again.")
        
        elif event.button.id == "quit_button":
            log.write("Exiting the application...")
            self.exit()

    async def add_site(self, command, log):
        try:
            parts = command.split()
            if len(parts) != 8:
                raise ValueError("Invalid format for add command. Expected 8 arguments.")
            _, moniker, region, router_ip, switch_ip, data_vlan, wireless_vlan, guest_vlan = parts
            now = datetime.now()
            self.cursor.execute("""
            INSERT INTO Sites (moniker, region, router_ip, switch_ip, data_vlan_id, wireless_vlan_id, guest_vlan_id, created_at, updated_at)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, (moniker, region, router_ip, switch_ip, int(data_vlan), int(wireless_vlan), int(guest_vlan), now, now))
            self.conn.commit()
            log.write(f"Record successfully added: {moniker}")
        except Exception as e:
            log.write(f"Error adding site: {e}. Please try again.")

    async def view_sites(self, command, log):
        try:
            if command != "view":
                raise ValueError("Invalid format for view command. No additional arguments expected.")
            self.cursor.execute("SELECT * FROM Sites")
            sites = self.cursor.fetchall()
            if sites:
                log.write("Displaying all sites:")
                for site in sites:
                    log.write(str(site))
                log.write("All records displayed successfully.")
            else:
                log.write("No records found in the database.")
        except Exception as e:
            log.write(f"Error: {e}. Please try again.")

    async def update_site(self, command, log):
        try:
            parts = command.split()
            if len(parts) != 4:
                raise ValueError("Invalid format for update command. Expected 4 arguments.")
            _, moniker, field, new_value = parts
            now = datetime.now()
            self.cursor.execute(f"UPDATE Sites SET {field} = ?, updated_at = ? WHERE moniker = ?", (new_value, now, moniker))
            if self.cursor.rowcount > 0:
                self.conn.commit()
                log.write(f"Record successfully updated: {moniker}")
            else:
                log.write(f"No matching record found for update: {moniker}")
        except Exception as e:
            log.write(f"Error updating site: {e}. Please try again.")

    async def delete_site(self, command, log):
        try:
            parts = command.split()
            if len(parts) != 2:
                raise ValueError("Invalid format for delete command. Expected 2 arguments.")
            _, moniker = parts
            self.cursor.execute("DELETE FROM Sites WHERE moniker = ?", (moniker,))
            if self.cursor.rowcount > 0:
                self.conn.commit()
                log.write(f"Record successfully deleted: {moniker}")
            else:
                log.write(f"No matching record found for deletion: {moniker}")
        except Exception as e:
            log.write(f"Error deleting site: {e}. Please try again.")

    def on_shutdown(self):
        self.conn.close()


if __name__ == "__main__":
    SiteManagerApp().run()
