
# Collection – This is an experimental repository for testing 
tasks/admin=>
from django.contrib import admin
from .models import Task

class TaskAdmin(admin.ModelAdmin):
	list_filter = ('completed', 'created_at', 'user')
	list_editable = ('completed',)
	search_fields = ('title', 'description')
	list_display = ['title', 'completed']

	class Meta:
		model = Task
	
admin.site.register(Task, TaskAdmin)

tasks/models.py

from django.utils.translation import gettext_lazy as _
from django.contrib.auth import get_user_model
from django.db import models

UserModel = get_user_model()

class Task(models.Model):

    user = models.ForeignKey(
        UserModel, 
        related_name='users', 
        on_delete=models.CASCADE, 
        verbose_name=_('User')
    )

    title = models.CharField(_('Title'), max_length=100)
    description = models.TextField(_('Description'))
    created_at = models.DateTimeField(_("Created at"), auto_now_add=True)
    updated_at = models.DateTimeField(_("Updated at"), auto_now=True)
    completed = models.BooleanField(_('Completed'), default=False)

    class Meta:
        ordering = ('title', )
        verbose_name = 'task'
        verbose_name_plural = 'tasks'
   
    def __str__(self):
        return str(self.title)
    

    tasks/views.py

    import csv
import xlwt
from xlwt import Workbook 
from .permissions import IsOwner
from rest_framework import viewsets, status
from django.http import HttpResponse
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .serializers import TaskSerializer
from .models import Task
        
from rest_framework.permissions import IsAuthenticated



class TaskView(viewsets.ModelViewSet):
    """
    This viewset automatically provides `list`, `create`, `retrieve`,
    `update` and `destroy` actions.
    """
    permission_classes = [IsAuthenticated, IsOwner]
    serializer_class = TaskSerializer
    queryset = Task.objects.all()


@api_view(['GET'])
def my_tasks(request):
    """
    List my all tasks.
    """
    tasks = Task.objects.filter(user=request.user)
    serializer = TaskSerializer(tasks, many=True)
    return Response(serializer.data)


@api_view(['GET'])
def task_detail(request, pk):
    """
    Get a specific task
    """
    try:
        task = Task.objects.get(user=request.user, pk=pk)
    except Task.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    serializer = TaskSerializer(task)
    return Response(serializer.data)



@api_view(['GET'])
def tasks_completed(request):
    """
    List my all completed tasks.
    """
    tasks = Task.objects.filter(
        user=request.user,
        completed=True
    )
    serializer = TaskSerializer(tasks, many=True)
    return Response(serializer.data)


@api_view(['GET'])
def tasks_incompleted(request):
    """
    List all incompleted tasks.
    """
    tasks = Task.objects.filter(
        user=request.user, 
        completed=False
    )
    serializer = TaskSerializer(tasks, many=True)
    return Response(serializer.data)



def save_as_csv(request):
    """
        Export data to csv file.
    """
    # Create the HttpResponse object with the appropriate CSV header.
    response = HttpResponse(content_type='text/csv')
    response['Content-Disposition'] = 'attachment; filename="tasks_rate.csv"'

    # Retrieve the username of the data owner.
    user_name = request.user.username

    # Number of completed tasks
    completed_tasks = Task.objects.filter(
        user=request.user, completed=True).count()

    # Number of incompleted tasks
    incompleted_tasks = Task.objects.filter(
        user=request.user, completed=False).count()

    # Number of all tasks
    all_tasks = Task.objects.filter(user=request.user).count()

    # Calculate the ratio between completed and incomplete tasks.
    rate = 100
    if incompleted_tasks != 0:
        rate = round((completed_tasks*100)/(all_tasks), 1)


    writer = csv.writer(response)

    # f-Strings: A New and Improved Way to Format Strings in Python>= 3.6
    writer.writerow(['Username', f'{user_name}'])
    writer.writerow([])
    writer.writerow(['All Tasks', 'Completed', 'Incompleted', 'Rate'])
    writer.writerow([f'{all_tasks}', f'{completed_tasks}', f'{incompleted_tasks}', f'%{rate}'])


    return response



def save_as_xls(request):
    """
        Export data to excel file.
    """

    response = HttpResponse(content_type='application/ms-excel')
    response['Content-Disposition'] = 'attachment; filename="tasks_rate.xls"'

    # Workbook is created 
    wb = Workbook() 

    # add_sheet is used to create sheet. 
    sheet1 = wb.add_sheet('Ratio') 

    name_style = xlwt.easyxf('font: bold 1, color red')
    style = xlwt.easyxf('font: bold 1')

    user_name = request.user.username

    sheet1.write(0, 0, 'Username', name_style) 
    sheet1.write(0, 1, f'{user_name}', name_style)

    # Heads
    sheet1.write(2, 0, 'All Tasks', style)
    sheet1.write(2, 1, 'Completed', style)
    sheet1.write(2, 2, 'Incompleted', style)
    sheet1.write(2, 3, 'Rate', style)

    completed_tasks = Task.objects.filter(
        user=request.user, completed=True).count()

    incompleted_tasks = Task.objects.filter(
        user=request.user, completed=False).count()

    all_tasks = Task.objects.filter(user=request.user).count()

    rate = 100
    if incompleted_tasks != 0:
        rate = round((completed_tasks*100)/(all_tasks), 1)

    sheet1.write(3, 0, f'{all_tasks}')
    sheet1.write(3, 1, f'{completed_tasks}')
    sheet1.write(3, 2, f'{incompleted_tasks}')
    sheet1.write(3, 3, f'%{rate}')

    # save the file.
    wb.save(response)

    return response

tasks/urls.py

from django.urls import path, include
from rest_framework import routers
from tasks import views

app_name = 'tasks'

# Create a router and register our viewsets with it.
router = routers.DefaultRouter()
router.register('', views.TaskView, 'tasks')

# The API URLs are now determined automatically by the router.
urlpatterns = [


    path('tasks/', include(router.urls)),

    path('me/all/', views.my_tasks, name='my-tasks'),
    path('me/<int:pk>/', views.task_detail, name='task-detail'),

    path('me/completed/', views.tasks_completed, name='compleated'),
    path('me/incompleted/', views.tasks_incompleted, name='incompleated'),

    # Export data.
    path('export/csv/', views.save_as_csv, name='save-as-csv'),
    path('export/xls/', views.save_as_xls, name='save-as-xls')

]

tasks/permission.py
from rest_framework import permissions

class IsOwner(permissions.BasePermission):
    """
        Custom permission to only allow owners of profile to view or edit it.
    """
    def has_object_permission(self, request, view, obj):
        return obj.user == request.user
    
    tasks/serializers.py
    
from rest_framework import serializers
from .models import Task
      
class TaskSerializer(serializers.ModelSerializer):
	class Meta:
		model = Task
		fields = ('user','id','title','description','completed','created_at','updated_at')


now users app files


users/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from .forms import CustomUserCreationForm, CustomUserChangeForm
from .models import CustomUser

@admin.register(CustomUser)
class CustomUserAdmin(UserAdmin):
    add_form = CustomUserCreationForm
    form = CustomUserChangeForm
    model = CustomUser

    list_display = ["id", "username", "email", "name", "is_staff", "is_active"]
    search_fields = ["username", "email", "name"]

    # show `name` on edit page
    fieldsets = UserAdmin.fieldsets + (
        (None, {"fields": ("name",)}),
    )
    # show `name` on add page
    add_fieldsets = UserAdmin.add_fieldsets + (
        (None, {"fields": ("name",)}),
    )



users/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class CustomUser(AbstractUser):
    name = models.CharField(max_length=255, blank=True)
    email = models.EmailField(unique=True)

    def __str__(self):
        return self.username

    
users/serializers.py

from django.contrib.auth import get_user_model
from rest_framework import serializers
from rest_framework.authtoken.models import Token

User = get_user_model()

class UserCreateSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ("username", "email", "password")

    def create(self, validated_data):
        user = User.objects.create_user(
            username=validated_data["username"],
            email=validated_data["email"],
            password=validated_data["password"],
        )
        # create token so /login will return it immediately
        Token.objects.create(user=user)
        return user

class UserListSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ("id", "username", "email", "name")

users/forms.py

from django.contrib.auth.forms import UserCreationForm, UserChangeForm
from .models import CustomUser

class CustomUserCreationForm(UserCreationForm):
    class Meta(UserCreationForm.Meta):
        model = CustomUser
        fields = ("username", "email", "name")

class CustomUserChangeForm(UserChangeForm):
    class Meta:
        model = CustomUser
        fields = ("username", "email", "name")

users/views.py

from django.contrib.auth import get_user_model
from rest_framework import generics, permissions
from .serializers import UserCreateSerializer, UserListSerializer

User = get_user_model()

class UserCreateView(generics.CreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserCreateSerializer
    permission_classes = [permissions.AllowAny]

class UserListView(generics.ListAPIView):
    queryset = User.objects.all().order_by("id")
    serializer_class = UserListSerializer
    permission_classes = [permissions.IsAuthenticated]

user/urls.py

from django.urls import path
from rest_framework.authtoken.views import obtain_auth_token
from .views import UserCreateView, UserListView

app_name = "users"

urlpatterns = [
    path("", UserListView.as_view(), name="user-list"),               # GET (auth)
    path("register/", UserCreateView.as_view(), name="user-create"),  # POST
    path("login/", obtain_auth_token, name="login"),                  # POST -> token
]

