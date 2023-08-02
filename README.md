"""
Tkinter Toolkit
Author: Akash Bora
Version: 0.2
License: MIT
Homepage: https://github.com/Akascape/tkinter-toolkit
"""

import customtkinter
from PIL import Image, ImageTk
import os
import json
from urllib.request import urlopen, urlretrieve
import io
import webbrowser
import sys
import pkg_resources

customtkinter.set_appearance_mode("System") 
customtkinter.set_default_color_theme("dark-blue")

class App(customtkinter.CTk):

    DIRPATH = os.path.join(os.path.dirname(__file__))
    LOADED_IMAGES = {}
    
    def __init__(self):
        super().__init__()
        
        self.title("Tkinter Toolkit")
        self.width = int(self.winfo_screenwidth()/2.5)
        self.height = int(self.winfo_screenheight()/2)
        self.geometry(f"{self.width}x{self.height}")
        self.minsize(500,500)
        self.bind("<1>", lambda event: event.widget.focus_set())
        self.iconpath = ImageTk.PhotoImage(file=os.path.join("assets","logo.png"))
        self.wm_iconbitmap()
        self.iconphoto(False, self.iconpath)
        
        self.frame = customtkinter.CTkFrame(master=self)
        self.frame.pack(expand=True, fill="both", padx=10, pady=10)

        self.frame.columnconfigure(1, weight=1)
        self.frame.rowconfigure(2, weight=1)
        self.font = customtkinter.ThemeManager.theme["CTkFont"]["family"]
   
        self.label = customtkinter.CTkLabel(master=self.frame, text="Tkinter Toolkit", font=(self.font,25,"bold"))
        self.label.grid(row=0, column=0, padx=20, pady=10)

        self.entry = customtkinter.CTkEntry(master=self.frame, placeholder_text="search", width=200)
        self.entry.grid(row=0, column=1, pady=10, sticky="e")
        self.entry.bind("<KeyRelease>", lambda e: self.search_package(self.entry.get()))
        
        self.about_button = customtkinter.CTkButton(master=self.frame, text="i", hover=False, width=30, command=self.open_about_window)
        self.about_button.grid(row=0, column=2, padx=10, pady=10, sticky="e")
        
        self.option_type = customtkinter.CTkSegmentedButton(self.frame, values=["All","pip", "manual"], command=self.filter_packages)
        self.option_type.set("All")
        self.option_type.grid(row=1, column=0, columnspan=3, padx=10, pady=10, sticky="ew")

        self.scrollable_frame = customtkinter.CTkScrollableFrame(self.frame)
        self.scrollable_frame.grid(row=2, column=0, columnspan=3, padx=10, pady=(0,10), sticky="nsew")
        
        self.ctkimage = customtkinter.CTkImage(Image.open(os.path.join(os.path.dirname(customtkinter.__file__),"assets","icons","CustomTkinter_icon_Windows.ico")))
        self.tkimage = customtkinter.CTkImage(Image.open(os.path.join(App.DIRPATH, "assets", "tk.png")))
        self.packageimage = customtkinter.CTkImage(Image.open(os.path.join(App.DIRPATH, "assets", "pkg.png")))
        
        self.item_frame = {}
        self.modules = [pkg.key for pkg in pkg_resources.working_set]
        self.read_database()
        
    def add_item(self, name, icon):
        """ add new package to the list """
        self.item_frame[name] = customtkinter.CTkFrame(self.scrollable_frame)
        self.item_frame[name].pack(expand=True, fill="x", padx=5, pady=5)
        
        if icon=="tk":
            icon = self.tkimage
        elif icon=="ctk":
            icon = self.ctkimage
        elif icon=="pkg":
            icon = self.packageimage
        else:
            icon = None

        self.item_frame[name].columnconfigure(0, weight=1)
        item_name = customtkinter.CTkButton(self.item_frame[name], fg_color="transparent", image=icon,
                                            text_color=customtkinter.ThemeManager.theme["CTkLabel"]["text_color"],
                                            height=50, anchor="w", font=(self.font, 15, "bold"), width=500,
                                            text=name, hover=False, command=lambda: self.open_info_window(name))
        item_name.grid(row=0, column=0, sticky="ew", pady=5, padx=5)
        
        if self.data[name]["name"] in self.modules:
            version = pkg_resources.get_distribution(self.data[name]["name"]).version
            desc = f"{self.data[name]['desc']} \nversion: {version}"
            self.data[name]["installation"] = f"{self.data[name]['installation']} --upgrade"
        else:
            self.item_frame[name].configure(fg_color=customtkinter.ThemeManager.theme["CTkButton"]["fg_color"])
            desc = f"{self.data[name]['desc']} "
         
        item_label = customtkinter.CTkLabel(self.item_frame[name], width=250, justify="left", text=desc, anchor="w", wraplength=250)
        item_label.grid(row=0, column=1, padx=5)
        
    def search_package(self, string):
        """ search the packages based on package tags """
        for i in self.data.keys():
            for j in self.data[i]["tags"]:
                if j.replace(" ", "").startswith(string.lower().replace(" ", "")):
                    if self.data[i]["type"]==self.option_type.get() or self.option_type.get()=="All":
                        self.item_frame[i].pack(expand=True, fill="x", padx=5, pady=5)
                        break
                else:
                    self.item_frame[i].pack_forget()
        self.scrollable_frame._parent_canvas.yview_moveto(0.0)
        
    def filter_packages(self, type_):
        """ filter out packages based on download type """
        if type_=="All":
            for i in self.item_frame.values():
                i.pack(expand=True, fill="x", padx=5, pady=5)
        elif type_=="pip":
            for i in self.item_frame.values():
                i.pack_forget()
            for i in self.data.keys():
                if self.data[i]["type"]=="pip":
                    self.item_frame[i].pack(expand=True, fill="x", padx=5, pady=5)
        elif type_=="manual":
            for i in self.item_frame.values():
                i.pack_forget()
            for i in self.data.keys():
                if self.data[i]["type"]=="manual":
                    self.item_frame[i].pack(expand=True, fill="x", padx=5, pady=5)
        self.search_package(self.entry.get())
        
    def open_about_window(self):
        """ open about window """
        def close_toplevel():
            about_window.destroy()
            self.about_button.configure(state="normal")
            
        def update_database():
            """ update the database and check for new packages """
            database_file = os.path.join(App.DIRPATH, "assets", "database.json")
            try:
                urlretrieve('https://raw.githubusercontent.com/Akascape/tkinter-toolkit/main/assets/database.json', database_file)
            except:            
                update_label.configure(text="no connection!")
                return
            for i in self.item_frame.values():
                i.pack_forget()
            self.item_frame = {}
            self.read_database()
            update_label.configure(text="database updated!")
            
        about_window = customtkinter.CTkToplevel(self)
        about_window.title("About")
        about_window.transient(self)
        about_window.geometry("300x200")
        about_window.resizable(False, False)
        about_window.protocol("WM_DELETE_WINDOW", close_toplevel)
        about_window.wm_iconbitmap()
        about_window.after(300, lambda: about_window.iconphoto(False, self.iconpath))
        
        label_title = customtkinter.CTkLabel(about_window, text="Tkinter Toolkit", anchor="w", font=(self.font,20,"bold"))
        label_title.pack(fill="x", padx=10, pady=10)
        info = "Package finder for tkinter and customtkinter. \n\nMade by Akascape"
        label_info = customtkinter.CTkLabel(about_window, text=info, anchor="w", justify="left")
        label_info.pack(fill="x", padx=10)
        
        repo_link = customtkinter.CTkLabel(about_window, text="Homepage", font=(self.font,13), text_color=["blue","cyan"])
        repo_link.pack(anchor="w", padx=10)
        
        repo_link.bind("<Button-1>", lambda event: webbrowser.open_new_tab("https://github.com/Akascape/tkinter-toolkit"))
        repo_link.bind("<Enter>", lambda event: repo_link.configure(font=(self.font,13,"underline"), cursor="hand2"))
        repo_link.bind("<Leave>", lambda event: repo_link.configure(font=(self.font,13), cursor="arrow"))

        update_database = customtkinter.CTkButton(about_window, text="Update Database", command=update_database)
        update_database.pack(pady=10)

        update_label = customtkinter.CTkLabel(about_window, text="check for new packages!")
        update_label.pack()
        self.about_button.configure(state="disabled")

    def get_image(self, name):
        """ download the image preview """
        try:
            if name not in self.LOADED_IMAGES:
                file = urlopen(self.data[name]["image_url"])
                raw_data = file.read()
                file.close()
                image = Image.open(io.BytesIO(raw_data))
                file_data = customtkinter.CTkImage(image, size=(self.height, self.height*image.size[1]/image.size[0]))
                self.LOADED_IMAGES[name] = file_data
            return self.LOADED_IMAGES[name]
        except:
            return None
        
    def open_info_window(self, name):
        """ open the detail panel for the package """

        toplevel = customtkinter.CTkToplevel(self)
        toplevel.title(name)
        toplevel.transient(self)  
        toplevel.geometry(f"{self.width}x{self.height}")
        toplevel.resizable(False, False)
        toplevel.wm_iconbitmap()
        toplevel.after(300, lambda: toplevel.iconphoto(False, self.iconpath))
        
        scrollable_info = customtkinter.CTkScrollableFrame(toplevel)
        scrollable_info.pack(fill="both", padx=10, pady=10, expand=True)
        
        image_label = customtkinter.CTkLabel(scrollable_info, fg_color=self.frame._fg_color, height=1, corner_radius=20, text="")
        image_label.pack(fill="both", padx=10, pady=10, expand=True)
        
        if self.data[name]["image_url"]:
            file_data = self.get_image(name)
            if file_data:
                image_label.configure(image=file_data)
        
        label_name = customtkinter.CTkLabel(scrollable_info, text=name, anchor="w", font=(self.font, 25, "bold"))
        label_name.pack(fill="x", padx=10)

        label_desc = customtkinter.CTkLabel(scrollable_info, text=self.data[name]["full_desc"], anchor="w", wraplength=self.width-100,
                                            justify="left", font=(self.font,14))
        label_desc.pack(fill="x", padx=10, pady=10)
        
        label_owner = customtkinter.CTkLabel(scrollable_info, text=f"Author: {self.data[name]['author']}", anchor="w", font=(self.font, 15, "bold"))
        label_owner.pack(fill="x", padx=10)
        
        frame_link = customtkinter.CTkFrame(scrollable_info, fg_color="transparent")
        self.repo = customtkinter.CTkLabel(frame_link, text="Repository Link: ", font=(self.font,14,"bold"))
        self.repo.pack(anchor="w", side="left")
        
        repo_link = customtkinter.CTkLabel(frame_link, text=self.data[name]["repo_url"], font=(self.font,14), text_color=["blue","cyan"])
        repo_link.pack(anchor="w", side="left")
        frame_link.pack(padx=10, anchor="w")
        
        repo_link.bind("<Button-1>", lambda event: webbrowser.open_new_tab(self.data[name]["repo_url"]))
        repo_link.bind("<Enter>", lambda event: repo_link.configure(font=(self.font,14,"underline"), cursor="hand2"))
        repo_link.bind("<Leave>", lambda event: repo_link.configure(font=(self.font,14), cursor="arrow"))
        
        features = "Features:"
        for i in self.data[name]["highlights"]:
            features = f"{features} \n      ▶ {i}"
            
        highlights = customtkinter.CTkLabel(scrollable_info, text=features, anchor="w", justify="left")
        highlights.pack(fill="x",padx=10)

        label_install = customtkinter.CTkLabel(scrollable_info, text=f"Installation type: {self.data[name]['type']}", anchor="w")
        label_install.pack(fill="x", padx=12, pady=5)
        
        entry_pip = customtkinter.CTkEntry(scrollable_info)
        entry_pip.pack(fill="x", padx=10, pady=(0,10))
        entry_pip.insert(0,self.data[name]["installation"])
        entry_pip.configure(state="readonly")
        
    def read_database(self):
        """ read the database containing package data """
        database = os.path.join(App.DIRPATH, "assets", "database.json")
        if os.path.exists(database):
            with open(database) as f:
                self.data = json.load(f)
        for i in self.data.keys():
            self.add_item(name=i, icon=self.data[i]["icon"])
            
if __name__ == "__main__":
    app = App()
    app.mainloop()
