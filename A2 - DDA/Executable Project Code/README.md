# Requirements
- Submit here the project code
- All resources like images, video/audio files required to run your project

import tkinter as tk
from tkinter import ttk
import requests
from PIL import Image, ImageTk
from io import BytesIO

API_KEY = "4ba37ce87bae7750b808ceda5703e5ab"
BASE_URL = "https://api.themoviedb.org/3"
IMAGE_BASE = "https://image.tmdb.org/t/p/w300"

class MovieApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Movie Application")
        self.geometry("950x650")
        self.configure(bg="black")
        
        #main container that holds all pages (frames)
        self.container = tk.Frame(self, bg="black")
        self.container.pack(fill="both", expand=True)
        
        #dictionary to store all page frames for easy access
        self.frames = {}
        
        #variables to track navigation state
        self.selected_movie = None
        self.last_page = None #remembers the previous page for proper back navigation
        self.current_page = None #keeps track of the currently displayed page
        
        #create instances of all three pages and place them in the container
        for F in (PageOne, PageTwo, PageThree):
            frame = F(self.container, self)
            self.frames[F] = frame
            frame.grid(row=0, column=0, sticky="nsew")
        
        #start the application by showing the first page 
        self.show_frame(PageOne)
    
    def show_frame(self, page):
        #before switching pages, save the current page as last_page for back button use
        if hasattr(self, "current_page") and page != self.current_page:
            self.last_page = self.current_page
        
        #update the current page reference
        self.current_page = page
        
        #retrieve the frame (page) from the dictionary
        frame = self.frames[page]
        
        #if navigating to the movie details page, load the selected movie data
        if page == PageThree:
            frame.load_movie()
        
        #bring the requested frame to the front (raise it above others)
        frame.tkraise()

class PageOne(tk.Frame):
    def __init__(self, parent, controller):
        super().__init__(parent, bg="black")
        self.controller = controller
        
        #center frame to position all elements in the middle of the screen
        center = tk.Frame(self, bg="black")
        center.place(relx=0.5, rely=0.5, anchor="center")
        
        #main title label split over two lines for visual appeal
        tk.Label(
            center,
            text="YOUR ENTERTAINMENT\nIN ONE PLACE",
            fg="red",
            bg="black",
            font=("Arial", 32, "bold"),
            justify="center"
        ).pack(pady=10)
        
        #subtitle explaining the purpose of the application
        tk.Label(
            center,
            text="Browse trending movies and explore details",
            fg="white",
            bg="black",
            font=("Arial", 16),
            justify="center"
        ).pack(pady=10)
        
        #button to move to the movie browsing page
        browse_btn = tk.Button(
            center,
            text="BROWSE",
            bg="red",
            fg="black",
            font=("Arial", 14, "bold"),
            width=15,
            command=lambda: controller.show_frame(PageTwo)
        )
        browse_btn.pack(pady=20)
        
        #keyboard accessibility: pressing enter while button is focused triggers browse
        browse_btn.focus_set()
        browse_btn.bind("<Return>", lambda e: controller.show_frame(PageTwo))

class PageTwo(tk.Frame):
    def __init__(self, parent, controller):
        super().__init__(parent, bg="black")
        self.controller = controller
        self.poster_images = [] #list to hold photoimage references to prevent garbage collection
        
        #top bar containing back button, search entry, and search button
        top_frame = tk.Frame(self, bg="black")
        top_frame.pack(pady=10, fill="x")
        
        #back button returns to the previous page (usually page one)
        self.back_btn = tk.Button(
            top_frame,
            text="<< Back",
            bg="red",
            fg="black",
            font=("Arial", 12),
            command=self.go_back
        )
        self.back_btn.pack(side="left", padx=10)
        
        #text entry field for typing movie search queries
        self.search_entry = tk.Entry(top_frame, width=50, font=("Arial", 12))
        self.search_entry.pack(side="left", padx=10)
        self.search_entry.bind("<Return>", lambda e: self.search_movies())
        
        #search button triggers movie search using the entered query
        search_btn = tk.Button(
            top_frame,
            text="Search",
            bg="red",
            fg="black",
            font=("Arial", 12),
            command=self.search_movies
        )
        search_btn.pack(side="left", padx=5)
        
        #animated scrolling banner displaying different movie categories
        self.banner = tk.Label(
            self,
            text="NOW PLAYING • TRENDING • TOP RATED",
            fg="white",
            bg="black",
            font=("Arial", 14)
        )
        self.banner.pack(pady=5)
        self.animate_banner()
        
        #frame that will hold the grid of movie posters and titles
        self.grid_frame = tk.Frame(self, bg="black")
        self.grid_frame.pack(pady=20)
        
        #load popular movies when the page is first created
        self.load_movies()
    
    def go_back(self):
        #navigate to the last visited page, defaulting to page one if none recorded
        target = self.controller.last_page if self.controller.last_page else PageOne
        self.controller.show_frame(target)
    
    def animate_banner(self):
        #creates a marquee-like effect by rotating text one character at a time
        text = self.banner.cget("text")
        self.banner.config(text=text[1:] + text[0])
        self.after(150, self.animate_banner)
    
    def fetch_movies(self, query=None):
        #retrieves movie data from tmdb api
        #if query is provided, performs a search; otherwise gets popular movies
        if query:
            url = f"{BASE_URL}/search/movie"
            params = {"api_key": API_KEY, "query": query}
        else:
            url = f"{BASE_URL}/movie/popular"
            params = {"api_key": API_KEY}
        
        #returns the first 6 results to keep the grid manageable
        return requests.get(url, params=params).json()["results"][:6]
    
    def load_movies(self, query=None):
        #clears previous movie widgets from the grid
        for widget in self.grid_frame.winfo_children():
            widget.destroy()
        self.poster_images.clear()
        
        #fetch movie list based on search query or default popular list
        movies = self.fetch_movies(query)
        
        #display each movie in a 3-column grid
        for i, movie in enumerate(movies):
            row, col = divmod(i, 3)
            poster_path = movie.get("poster_path")
            
            #skip movies without a poster image
            if not poster_path:
                continue
            
            #download and process the poster image
            img_data = requests.get(IMAGE_BASE + poster_path).content
            img = Image.open(BytesIO(img_data)).resize((180, 270))
            poster = ImageTk.PhotoImage(img)
            self.poster_images.append(poster) #keep reference to avoid garbage collection
            
            #button with poster image that opens details when clicked
            btn = tk.Button(
                self.grid_frame,
                image=poster,
                bg="black",
                bd=0,
                command=lambda m=movie: self.open_movie(m)
            )
            btn.grid(row=row * 2, column=col, padx=15, pady=5)
            
            #movie title label below the poster
            tk.Label(
                self.grid_frame,
                text=movie["title"],
                fg="white",
                bg="black",
                wraplength=180
            ).grid(row=row * 2 + 1, column=col, pady=5)
    
    def search_movies(self):
        #retrieves the text from the search entry and reloads movies with that query
        query = self.search_entry.get()
        self.load_movies(query)
    
    def open_movie(self, movie):
        #stores the selected movie in the main app and switches to details page
        self.controller.selected_movie = movie
        self.controller.show_frame(PageThree)

class PageThree(tk.Frame):
    def __init__(self, parent, controller):
        super().__init__(parent, bg="black")
        self.controller = controller
        self.poster_images = [] #holds references to additional images to prevent garbage collection
        
        #top bar with back button
        top_frame = tk.Frame(self, bg="black")
        top_frame.pack(fill="x", pady=5)
        
        self.back_btn = tk.Button(
            top_frame,
            text="<< Back",
            bg="red",
            fg="black",
            font=("Arial", 12),
            command=self.go_back
        )
        self.back_btn.pack(side="left", padx=10)
        
        #movie title display
        self.title_label = tk.Label(self, fg="red", bg="black",
                                    font=("Arial", 24, "bold"))
        self.title_label.pack(pady=10)
        
        #frame for star rating display
        self.rating_frame = tk.Frame(self, bg="black")
        self.rating_frame.pack(pady=5)
        
        #movie overview/description text
        self.desc = tk.Label(self, fg="white", bg="black",
                             wraplength=900, justify="left")
        self.desc.pack(pady=10)
        
        #section for additional images, cast, and review
        self.additional_frame = tk.Frame(self, bg="black")
        self.additional_frame.pack(pady=10)
    
    def go_back(self):
        #returns to the previous page (usually page two)
        target = self.controller.last_page if self.controller.last_page else PageTwo
        self.controller.show_frame(target)
    
    def create_stars(self, rating):
        #converts tmdb vote average (out of 10) to a 5-star display
        #clears previous stars first
        for widget in self.rating_frame.winfo_children():
            widget.destroy()
        
        stars = int(round(rating / 2)) #divide by 2 to map 10-point scale to 5 stars
        
        #display filled stars
        for i in range(stars):
            tk.Label(self.rating_frame, text="★", fg="yellow", bg="black", font=("Arial", 18)).pack(side="left")
        
        #display empty stars to complete the 5-star row
        for i in range(5 - stars):
            tk.Label(self.rating_frame, text="☆", fg="yellow", bg="black", font=("Arial", 18)).pack(side="left")
    
    def load_movie(self):
        #populates the details page with information about the selected movie
        movie = self.controller.selected_movie
        
        #update title and description
        self.title_label.config(text=movie["title"])
        self.desc.config(text=movie.get("overview", "No description available"))
        
        #display rating as stars
        rating = movie.get("vote_average", 0)
        self.create_stars(rating)
        
        #clear previous additional content
        for widget in self.additional_frame.winfo_children():
            widget.destroy()
        self.poster_images.clear()
        
        movie_id = movie["id"]
        
        #fetch additional posters/backdrops for the movie
        img_data = requests.get(f"{BASE_URL}/movie/{movie_id}/images", params={"api_key": API_KEY}).json()
        posters = img_data.get("posters", [])[:3]
        
        for poster in posters:
            path = poster.get("file_path")
            if not path:
                continue
            img = Image.open(BytesIO(requests.get(IMAGE_BASE + path).content)).resize((200, 300))
            poster_img = ImageTk.PhotoImage(img)
            self.poster_images.append(poster_img)
            tk.Label(self.additional_frame, image=poster_img, bg="black").pack(side="left", padx=5)
        
        #fetch top 3 cast members
        cast_data = requests.get(f"{BASE_URL}/movie/{movie_id}/credits", params={"api_key": API_KEY}).json()
        cast_list = [c['name'] for c in cast_data.get("cast", [])[:3]]
        tk.Label(self.additional_frame, text=f"Cast: {', '.join(cast_list)}", fg="white", bg="black",
                 font=("Arial", 12)).pack(pady=5)
        
        #fetch one review if available
        review_data = requests.get(f"{BASE_URL}/movie/{movie_id}/reviews", params={"api_key": API_KEY}).json()
        reviews = review_data.get("results", [])
        if reviews:
            review_text = reviews[0]["content"][:250] + "..."
            tk.Label(self.additional_frame, text=f"Review: {review_text}", fg="white", bg="black",
                     wraplength=900, justify="left").pack(pady=5)

if __name__ == "__main__":
    app = MovieApp()
    app.mainloop()