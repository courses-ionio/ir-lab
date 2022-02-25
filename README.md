## IR-lab: Εργαστήριο ανάκτησης πληροφορίας

    Εργαστήριο Γαληνός: Παρασκευή, 11:00πμ

---

### Web scrapping (python)

Θα αξιοποιήσουμε [jupyter lab](https://jupyter.org/) για την υλοποίηση ενός απλού web scrapper.  
Θα κάνουμε αξιοποίηση στο μηχάνημά μας και σε google colab.

* Εγκατάσταση jupyter lab σε ένα docker container (για να είναι disposable :-), μπορείτε όμως να <span style="background-color: #FFFFA0">προσπεράσετε τη χρήση docker (αγνοήστε το κίτρινο highlight)</span> και να κάνετε native εγκατάσταση στο μηχάνημά/λειτουργικό σας.
    * <span style="background-color: #FFFFA0">Δημιουργία ενός python container  
    ```docker run --name jupyter -ti -p 8888:8888 -v `pwd`:/jupyter python:latest /bin/bash```  
    Το `-p` θα μας παρέχει πρόσβαση στο web server που θα ξεκινήσει εντός του container και θα _ακούει_ στο port `8888`.  
    Το `-v` κάνει map το τρέχον directory στο `/jupyter` εντός του container (αυτό το directory θα είναι διαθέσιμο και στο host λειτουργικό μας και μέσα στον container)</span>
    * Εγκατάσταση του jupyter lab  
    ```pip install jupyterlab```
    * Εκκίνηση του jupyter lab  
    ```jupyter-lab --ip 0.0.0.0 --port 8888 --allow-root```
    * Ελέγξτε στο terminal για το `token` το οποίο σας δίνει πρόσβαση στο jupyter lab, πχ:
    ```
    [C 2022-02-24 04:48:56.534 ServerApp]

    To access the server, open this file in a browser:
        file:///root/.local/share/jupyter/runtime/jpserver-27-open.html
    Or copy and paste one of these URLs:
        http://de00f90ffb61:8888/lab?token=fe8650eb49bdace494ab617f7b922535f33758fea99c824c
     or http://127.0.0.1:8888/lab?token=fe8650eb49bdace494ab617f7b922535f33758fea99c824c
    ```
    Χρησιμοποιήστε το url τύπου `http://127.0.0.1:8888/lab?token=...`
    * Καλώς ήρθατε στο lab σας:  
    ![Jupyter Lab](./_img/your-jupyter-lab.jpg)
    * Γνωρίστε καλύτερα το jupyter lab:
        * https://www.dataquest.io/blog/jupyter-notebook-tutorial/
        * https://docs.jupyter.org/en/latest/

* Δημιουργήστε ένα `Python 3 notebook` και ξεκινάμε για το web scrapping
    * Μελετήστε τον κώδικα που περιγράφει πώς κάνουμε web scrapping με χρήση της βιβλιοθήκης Beautiful Soup της python: https://realpython.com/python-web-scraping-practical-introduction/#use-an-html-parser-for-web-scraping-in-python  
    * Δημιουργήστε ένα notebook στο οποίο να κάνετε scrap τα έργα του [Shakespeare](http://shakespeare.mit.edu/), αποθηκεύοντας κάθε έργο σε μορφή απλού κειμένου σε ένα ξεχωριστό αρχείο με όνομα τον τίτλο του έργου. Αναμίξτε τον κώδικα python που γράφετε με block κειμένου markdown στα οποία εξηγείτε τι κάνετε σε κάθε (ουσιώδες) βήμα.
    * Happy coding :-)
    * Όταν ολοκληρώσετε το notebook σας, αποθηκεύστε το τοπικά στον υπολογιστή σας (Download) και στη συνέχεια ανεβάστε το στο https://colab.research.google.com/. Δοκιμάστε να εκτελέσετε εκεί το notebook που φτιάξατε στο δικό σας jupyter lab.
