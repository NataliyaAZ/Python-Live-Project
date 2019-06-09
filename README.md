The Python Live Project was one of the four projects I participated at the Tech Academy. During the project I was working in a team of several developers on creating a web scraper application in Django. Each of us could select as front end so back end user stories to work on. For me, the really great part about the project was getting rewarding experience with Django framework in a team of other passionate developers where we could share problems we faced and how we overcame them. 

My contributions to the project were two back end stories described below.

1.	User Profile Image in Navigation Bar

Once users create their accounts on the website the user profiles created too. The users then can upload their profile images and the image should appear in navigation bar.

models.py

#User Profile Model (including Profile Image column, 2 other fields were creating for user profile future extension)
class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    bio = models.TextField(max_length=500, blank=True)
    city = models.CharField(max_length=30, blank=True)
    image = models.ImageField(upload_to='profile_image',null=True, blank=True)

#Create UserProfile when  users sign up

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        UserProfile.objects.create(user=instance)

@receiver(post_save, sender=User)
def save_user_profile(sender, instance, **kwargs):
    instance.userprofile.save()
	
forms.py

class ProfileForm(forms.ModelForm):
    class Meta:
        model = UserProfile
        fields = ['image']

views.py

class ProfileView(TemplateView):
    template_name = 'profile.html'

    def get(self,request):
        form = ProfileForm()
        return render (request, self.template_name, {'form':form})

    def post(self, request):
         if request.method == 'POST':
         	form_old = ProfileForm(data=request.POST, files=request.FILES)
              if form_old.is_valid() and 'image' in request.FILES:
            	    request.user.userprofile.image.delete()

             form = ProfileForm(request.POST, request.FILES, instance=request.user.userprofile)
             if form.is_valid():
                  form.save()
        else:
            form = ProfileForm(instance=request.user.userprofile)
        return render (request, self.template_name, {'form':form})

.html (form for image upload)

<form style='color:white;font-size: 13px;' method="post" enctype = "multipart/form-data">
              <h3>You can upload or change Profile Image here:</h3>
               {% csrf_token %}
               {{ form }}
              <button type="submit">Upload</button>
</form>
.html (navigation bar)
<a class="navbar-brand" href="#">
      {% if user.userprofile.image %}
           <img src="{{ user.userprofile.image.url }}" width="50" height="48" alt="">
      {% endif %}
 </a>

2.	Create and Update Event in Google Events Calendar

While logged in, users should be able to create and update events in their google calendar. To create events users should enter event title, start date and time, and end date and time of the event. To update events users should first search for the event, then update and submit the updated event.

Forms.py 


class EventForm(forms.Form):
    event_title = forms.CharField(max_length=100,required=True)
    start_date = forms.DateField(initial=datetime.now(), required=True)
    start_time = forms.TimeField(initial=datetime.now(), required=True)
    end_date = forms.DateField(initial=datetime.now(), required=True)
    end_time = forms.TimeField(initial=datetime.now(), required=True)

class SearchForm(forms.Form):
    event_date = forms.DateField(initial=datetime.now(), required=True)
    event_time = forms.TimeField(initial=datetime.now(), required=True)
    
class UpdateForm(forms.Form):
    event_title_to_update = forms.CharField(max_length=100,required=True)
    event_startdate_to_update = forms.DateField(required=True)
    event_starttime_to_update = forms.TimeField(required=True)
    event_enddate_to_update = forms.DateField(required=True)
    event_endtime_to_update = forms.TimeField(required=True)
    event_id_donot_change = forms.CharField(max_length=100,required=True)

views.py

class EventCalendar(TemplateView):
    template_name = 'events-calendar.html'
     
    def get(self,request):
        form = EventForm()
        formSearch = SearchForm()
        formUpdate = UpdateForm()
        return render (request,self.template_name,{'form':form,'formSearch':formSearch,'formUpdate':formUpdate})
    
    #Connect to Google Calendar
    def getCalendarService(self):
        # Connect to Google calendar API
        SCOPES = ['https://www.googleapis.com/auth/calendar.events']
            
        creds = None
        # The file token.pickle stores the user's access and refresh tokens, and is
        # created automatically when the authorization flow completes for the first
        # time.
        if os.path.exists('token.pickle'):
            with open('token.pickle', 'rb') as token:
                creds = pickle.load(token)
        # If there are no (valid) credentials available, let the user log in.
        if not creds or not creds.valid:
            if creds and creds.expired and creds.refresh_token:
                creds.refresh(Request())
            else:
                flow = InstalledAppFlow.from_client_secrets_file(
                    'client_secret.json', SCOPES)
                creds = flow.run_local_server()
            # Save the credentials for the next run
            with open('token.pickle', 'wb') as token:
                pickle.dump(creds, token)

        return build('calendar', 'v3', credentials=creds)


    #Processing events
    def post(self, request):
                
        #Create Event
        if request.method == 'POST' and 'btnsubmit' in request.POST:
            form = EventForm(request.POST)
            if form.is_valid():
                data = request.POST.copy()
                event_summary = data.get('event_title')
                event_start_date = data.get('start_date')
                event_start_time = data.get('start_time')
                event_end_date = data.get('end_date')
                event_end_time = data.get('end_time')
                
                service = self.getCalendarService()

                #Get the event from the Event Form and insert into Google Calendar
                event = {
                    'summary': event_summary,
                    'start': {
                        'dateTime': event_start_date+'T'+event_start_time+'-07:00',
                        'timeZone': 'America/Los_Angeles',
                    },
                    'end': {
                    'dateTime': event_end_date+'T'+event_end_time+'-07:00',
                    'timeZone': 'America/Los_Angeles',
                    },
                    'reminders': {
                        'useDefault': False,
                        'overrides': [
                        {'method': 'email', 'minutes': 24 * 60},
                        {'method': 'popup', 'minutes': 10},
                        ],
                    },
                }

                event = service.events().insert(calendarId='primary', body=event).execute()
             
                return HttpResponseRedirect('/events-calendar/') 
        
        #Update events

        #Search for the event to update
        if request.method == 'POST' and 'btnsearch' in request.POST:
            formSearch = SearchForm(request.POST)
            if formSearch.is_valid():
                data = request.POST.copy()
                event_start_date = data.get('event_date')
                event_start_time = data.get('event_time')
                newVar=(chr(ord(event_start_time[7])+1))
                time_max = list(event_start_time)
                time_max[7] = newVar
                event_time_max = ''.join(str(t) for t in time_max)

                service = self.getCalendarService()

                #Find the event in Google calendar and display Event Title and Event Id in Update Event Form  
                events = service.events().list(calendarId='primary', timeMax=event_start_date+'T'+event_time_max+'-07:00',                                        timeMin=event_start_date+'T'+event_start_time+'-07:00').execute()
                event, = events['items']
                event_name = event['summary']
                event_id = event['id']
                formUpdate = UpdateForm(initial={'event_title_to_update':event_name, 'event_id_donot_change':event_id})
                formSearch = SearchForm()
                form = EventForm()
                
                return render (request, self.template_name, {'form':form,'formSearch':formSearch,'formUpdate':formUpdate})
        
 
                      
        #Submit updated event
        if  request.method == 'POST' and 'btnupdate' in request.POST:
            form = UpdateForm(request.POST)
            if form.is_valid():
                data = request.POST.copy()
                event_summary = data.get('event_title_to_update')
                event_start_date = data.get('event_startdate_to_update')
                event_start_time = data.get('event_starttime_to_update')
                event_end_date = data.get('event_enddate_to_update')
                event_end_time = data.get('event_endtime_to_update')
                event_id =  data.get('event_id_donot_change')

                service = self.getCalendarService()

                #Retrieve the event from Google Calendar, update, and submit 
                event = service.events().get(calendarId='primary', eventId=event_id).execute()

                event = {
                    'summary': event_summary,
                    'start': {
                        'dateTime': event_start_date+'T'+event_start_time+'-07:00',
                    },
                    'end': {
                    'dateTime': event_end_date+'T'+event_end_time+'-07:00',
                    },
                    'id': event_id,
                }
                
                event = service.events().update(calendarId='primary', eventId=event['id'], body=event).execute()
                print(event)
        return HttpResponseRedirect('/events-calendar/')

.html( extract from event-calendar.html - Create Form as an example)

<form  method="POST">
    {% csrf_token %}
        <div class="field has-addons">
             <div class="control is-expanded">
                  {{ form.as_ul }}
             </div>
       	     <div class="control">
                <button type="submit" name="btnsubmit" class="button is-info">
                     Submit
                </button>
             </div>
        </div>
 </form>

Working first time with API was quite challenging but highly beneficial. The main challenge was every API has its own information that it releases to an outside application and its own methods that should be used to retrieve those information. Thus, carefully go through the API documentation and obtaining API key can take a while.

Overall, the project was a great experience in using Django and Python for the web development.


