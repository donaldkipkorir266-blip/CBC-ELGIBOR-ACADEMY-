# CBC-ELGIBOR-ACADEMY-cbcrms/
├─ manage.py
├─ requirements.txt
├─ Dockerfile (optional)
├─ cbcrms/
│  ├─ __init__.py
│  ├─ settings.py
│  ├─ urls.py
│  ├─ wsgi.py
│  └─ asgi.py
├─ cbc/
│  ├─ __init__.py
│  ├─ admin.py
│  ├─ apps.py
│  ├─ models.py
│  ├─ views.py
│  ├─ api_views.py
│  ├─ serializers.py
│  ├─ forms.py
│  ├─ urls.py
│  ├─ utils.py
│  ├─ reports.py
│  ├─ templates/
│  │  └─ cbc/
│  │     ├─ base.html
│  │     ├─ dashboard.html
│  │     ├─ competency_entry.html
│  │     └─ report_card.html
│  └─ static/
│     └─ cbc/
│        ├─ js/
│        └─ css/
└─ docs/
   └─ student_template.xlsx  (example Excel)
   Django>=4.2
djangorestframework>=3.14
django-import-export>=3.1.1
Pillow>=9.0
weasyprint>=57.0   # optional: needs system libs
openpyxl>=3.0
pandas>=2.0
psycopg2-binary>=2.9  # for PostgreSQL
gunicorn>=20.1.0
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
# cbcrms/settings.py (partial)
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent
SECRET_KEY = 'replace-with-secure-secret'  # set via env in production
DEBUG = True  # set False in production
ALLOWED_HOSTS = ['localhost', '127.0.0.1', '.yourdomain.com']

INSTALLED_APPS = [
    'django.contrib.admin','django.contrib.auth','django.contrib.contenttypes',
    'django.contrib.sessions','django.contrib.messages','django.contrib.staticfiles',
    'rest_framework','import_export',
    'cbc',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
]

ROOT_URLCONF = 'cbcrms.urls'

TEMPLATES = [{
    'BACKEND':'django.template.backends.django.DjangoTemplates',
    'DIRS':[BASE_DIR / 'cbc' / 'templates'],
    'APP_DIRS': True,
    'OPTIONS': {'context_processors': [
        'django.template.context_processors.debug','django.template.context_processors.request',
        'django.contrib.auth.context_processors.auth','django.contrib.messages.context_processors.messages'
    ]},
}]

WSGI_APPLICATION = 'cbcrms.wsgi.application'

# Database - Use SQLite for dev, Postgres for prod
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',  # change to postgresql in production
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# Static & Media
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

# Authentication
LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = '/accounts/login/'

# Timezone
TIME_ZONE = 'Africa/Nairobi'
USE_TZ = True

# Email (configure in production)
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
# cbc/models.py
from django.db import models
from django.contrib.auth.models import User
from django.utils import timezone

ROLE_CHOICES = (('admin','Admin'),('teacher','Teacher'),('parent','Parent'))
DESCRIPTOR_CHOICES = (('EE','Emerging'),('ME','Meeting'),('AE','Above Expected'),('BE','Below Expected'))
AREA_CHOICES = (('Knowledge','Knowledge'),('Skills','Skills'),('Values','Values'))
LEVEL_CHOICES = (
    ('PP1','PP1'),('PP2','PP2'),
    ('G1','Grade 1'),('G2','Grade 2'),('G3','Grade 3'),
    ('G4','Grade 4'),('G5','Grade 5'),('G6','Grade 6'),
    ('J7','Grade 7'),('J8','Grade 8'),('J9','Grade 9'),
)

class School(models.Model):
    name = models.CharField(max_length=255)
    alias = models.CharField(max_length=50, blank=True)
    address = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    def __str__(self): return self.name

class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='profile')
    school = models.ForeignKey(School, on_delete=models.CASCADE, null=True, blank=True)
    role = models.CharField(max_length=20, choices=ROLE_CHOICES)
    phone = models.CharField(max_length=20, blank=True)
    def __str__(self): return f"{self.user.get_full_name() or self.user.username} ({self.role})"

class Classroom(models.Model):
    school = models.ForeignKey(School, on_delete=models.CASCADE)
    name = models.CharField(max_length=100)
    level = models.CharField(max_length=10, choices=LEVEL_CHOICES)
    form_teacher = models.ForeignKey(Profile, null=True, blank=True, on_delete=models.SET_NULL, related_name='form_classes')
    def __str__(self): return f"{self.name} - {self.school.name}"

class Student(models.Model):
    school = models.ForeignKey(School, on_delete=models.CASCADE)
    admission_number = models.CharField(max_length=50, unique=True)
    first_name = models.CharField(max_length=120)
    last_name = models.CharField(max_length=120)
    dob = models.DateField(null=True, blank=True)
    gender = models.CharField(max_length=20, blank=True)
    guardians = models.ManyToManyField(Profile, blank=True, related_name='children')
    photo = models.ImageField(upload_to='students', null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    def __str__(self): return f"{self.admission_number} {self.first_name} {self.last_name}"

class Enrollment(models.Model):
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='enrollments')
    classroom = models.ForeignKey(Classroom, on_delete=models.CASCADE, related_name='enrollments')
    year = models.IntegerField()
    start_date = models.DateField(default=timezone.now)
    end_date = models.DateField(null=True, blank=True)
    class Meta:
        unique_together = ('student','classroom','year')

class Term(models.Model):
    school = models.ForeignKey(School, on_delete=models.CASCADE)
    name = models.CharField(max_length=120)
    start_date = models.DateField()
    end_date = models.DateField()
    def __str__(self): return f"{self.name}"

class Subject(models.Model):
    school = models.ForeignKey(School, on_delete=models.CASCADE)
    name = models.CharField(max_length=150)
    code = models.CharField(max_length=50, blank=True)
    order = models.IntegerField(default=0)
    def __str__(self): return self.name

class Strand(models.Model):
    subject = models.ForeignKey(Subject, on_delete=models.CASCADE, related_name='strands')
    name = models.CharField(max_length=200)
    order = models.IntegerField(default=0)
    def __str__(self): return f"{self.subject.name} — {self.name}"

class SubStrand(models.Model):
    strand = models.ForeignKey(Strand, on_delete=models.CASCADE, related_name='substrands')
    name = models.CharField(max_length=200)
    order = models.IntegerField(default=0)
    area = models.CharField(max_length=20, choices=AREA_CHOICES)
    def __str__(self): return f"{self.strand.name} — {self.name}"

class Competency(models.Model):
    substrand = models.ForeignKey(SubStrand, on_delete=models.CASCADE, related_name='competencies')
    code = models.CharField(max_length=50, blank=True)
    title = models.CharField(max_length=255)
    description = models.TextField(blank=True)
    order = models.IntegerField(default=0)
    def __str__(self): return f"{self.code or ''} {self.title}"

class GradingScale(models.Model):
    school = models.ForeignKey(School, on_delete=models.CASCADE)
    name = models.CharField(max_length=120)
    descriptor = models.CharField(max_length=3, choices=DESCRIPTOR_CHOICES)
    min_score = models.FloatField()
    max_score = models.FloatField()
    class Meta:
        unique_together = ('school','name','descriptor')

class Assessment(models.Model):
    school = models.ForeignKey(School, on_delete=models.CASCADE)
    term = models.ForeignKey(Term, on_delete=models.CASCADE)
    classroom = models.ForeignKey(Classroom, on_delete=models.CASCADE)
    subject = models.ForeignKey(Subject, on_delete=models.CASCADE, null=True, blank=True)
    name = models.CharField(max_length=255)
    date = models.DateField(default=timezone.now)
    teacher = models.ForeignKey(Profile, null=True, blank=True, on_delete=models.SET_NULL)
    created_at = models.DateTimeField(auto_now_add=True)
    def __str__(self): return f"{self.name} ({self.classroom})"

class AssessmentItem(models.Model):
    assessment = models.ForeignKey(Assessment, on_delete=models.CASCADE, related_name='items')
    student = models.ForeignKey(Student, on_delete=models.CASCADE, related_name='assessment_items')
    competency = models.ForeignKey(Competency, on_delete=models.CASCADE)
    descriptor = models.CharField(max_length=3, choices=DESCRIPTOR_CHOICES, null=True, blank=True)
    score = models.FloatField(null=True, blank=True)
    remark = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    class Meta:
        unique_together = ('assessment','student','competency')

class Attendance(models.Model):
    classroom = models.ForeignKey(Classroom, on_delete=models.CASCADE)
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    date = models.DateField()
    present = models.BooleanField(default=True)
    note = models.TextField(blank=True)
    class Meta:
        unique_together = ('classroom','student','date')

class DisciplineNote(models.Model):
    classroom = models.ForeignKey(Classroom, on_delete=models.CASCADE)
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    date = models.DateField(default=timezone.now)
    note = models.TextField()
    action_taken = models.TextField(blank=True)
    reported_by = models.ForeignKey(Profile, null=True, blank=True, on_delete=models.SET_NULL)

class ClassTeacherNotes(models.Model):
    classroom = models.ForeignKey(Classroom, on_delete=models.CASCADE)
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    term = models.ForeignKey(Term, on_delete=models.CASCADE)
    strengths = models.TextField(blank=True)
    areas_for_improvement = models.TextField(blank=True)
    overall_remark = models.TextField(blank=True)
    class Meta:
        unique_together = ('classroom','student','term')

class ReportPDF(models.Model):
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    term = models.ForeignKey(Term, on_delete=models.CASCADE)
    file = models.FileField(upload_to='reports/')
    generated_by = models.ForeignKey(Profile, null=True, blank=True, on_delete=models.SET_NULL)
    generated_at = models.DateTimeField(auto_now_add=True)
    # cbc/admin.py
from django.contrib import admin
from import_export import resources
from import_export.admin import ImportExportModelAdmin
from .models import Student, School, Profile, Classroom, Subject, Strand, SubStrand, Competency, Assessment, AssessmentItem, Term, GradingScale

class StudentResource(resources.ModelResource):
    class Meta:
        model = Student
        fields = ('id','admission_number','first_name','last_name','dob','gender','school__name')

@admin.register(Student)
class StudentAdmin(ImportExportModelAdmin):
    resource_class = StudentResource
    list_display = ('admission_number','first_name','last_name','school')

admin.site.register(School)
admin.site.register(Profile)
admin.site.register(Classroom)
admin.site.register(Subject)
admin.site.register(Strand)
admin.site.register(SubStrand)
admin.site.register(Competency)
admin.site.register(Assessment)
admin.site.register(AssessmentItem)
admin.site.register(Term)
admin.site.register(GradingScale)
# cbc/forms.py
from django import forms
from .models import Assessment, AssessmentItem, DESCRIPTOR_CHOICES

class AssessmentCreateForm(forms.ModelForm):
    class Meta:
        model = Assessment
        fields = ['term','classroom','subject','name','date']

class AssessmentItemForm(forms.ModelForm):
    descriptor = forms.ChoiceField(choices=DESCRIPTOR_CHOICES, required=False)
    score = forms.FloatField(required=False)
    remark = forms.CharField(widget=forms.Textarea, required=False)
    class Meta:
        model = AssessmentItem
        fields = ['competency','student','descriptor','score','remark']
        # cbc/serializers.py
from rest_framework import serializers
from .models import Assessment, AssessmentItem, Student, Classroom, Competency

class CompetencySerializer(serializers.ModelSerializer):
    class Meta:
        model = Competency
        fields = '__all__'

class AssessmentItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = AssessmentItem
        fields = '__all__'

class AssessmentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Assessment
        fields = '__all__'
        # cbc/utils.py
from django.db.models import Avg, Count
from .models import GradingScale, DESCRIPTOR_CHOICES, AssessmentItem

def score_to_descriptor(school, score, grading_name='Default'):
    if score is None:
        return None
    scales = GradingScale.objects.filter(school=school, name=grading_name)
    for s in scales:
        if s.min_score <= score <= s.max_score:
            return s.descriptor
    return None

def student_area_summary(student, term):
    items = AssessmentItem.objects.filter(student=student, assessment__term=term)
    res = {}
    for area in ['Knowledge','Skills','Values']:
        area_items = items.filter(competency__substrand__area=area)
        avg = area_items.aggregate(avg_score=Avg('score'))['avg_score'] or 0
        counts = {d: area_items.filter(descriptor=d).count() for d,_ in DESCRIPTOR_CHOICES}
        res[area] = {'avg_score': avg, 'counts': counts}
    return res
    # cbc/views.py
from django.shortcuts import render, get_object_or_404, redirect
from django.contrib.auth.decorators import login_required
from .models import Classroom, Enrollment, Assessment, Competency, AssessmentItem, Term, Student, ReportPDF
from .forms import AssessmentCreateForm
from .utils import student_area_summary
from .reports import render_report_pdf_response

@login_required
def dashboard(request):
    profile = request.user.profile
    if profile.role == 'admin':
        classes = Classroom.objects.filter(school=profile.school)
    elif profile.role == 'teacher':
        classes = Classroom.objects.filter(form_teacher=profile)
    else:
        # parent: list children
        children = profile.children.all()
        return render(request, 'cbc/dashboard.html', {'children': children, 'profile': profile})
    return render(request, 'cbc/dashboard.html', {'classes': classes, 'profile': profile})

@login_required
def class_dashboard(request, class_id):
    classroom = get_object_or_404(Classroom, id=class_id)
    enrollments = classroom.enrollments.filter(end_date__isnull=True)
    students = [e.student for e in enrollments]
    terms = Term.objects.filter(school=classroom.school).order_by('-start_date')
    return render(request, 'cbc/class_dashboard.html', {'classroom': classroom, 'students': students, 'terms': terms})

@login_required
def create_assessment(request, class_id):
    classroom = get_object_or_404(Classroom, id=class_id)
    if request.method == 'POST':
        form = AssessmentCreateForm(request.POST)
        if form.is_valid():
            assessment = form.save(commit=False)
            assessment.school = request.user.profile.school
            assessment.teacher = request.user.profile
            assessment.classroom = classroom
            assessment.save()
            return redirect('cbc:competency_entry', assessment.id)
    else:
        form = AssessmentCreateForm(initial={'classroom': classroom})
    return render(request, 'cbc/create_assessment.html', {'form': form, 'classroom': classroom})

@login_required
def competency_entry(request, assessment_id):
    assessment = get_object_or_404(Assessment, id=assessment_id)
    enrollments = assessment.classroom.enrollments.filter(end_date__isnull=True)
    students = [e.student for e in enrollments]
    competencies = Competency.objects.filter(substrand__strand__subject=assessment.subject).order_by('order') if assessment.subject else Competency.objects.filter(substrand__strand__subject__school=assessment.school).order_by('substrand__strand__subject','order')
    if request.method == 'POST':
        # expecting a JSON payload or form fields for items
        # for simplicity we accept form fields: item-<studentid>-<competencyid>-descriptor, -score, -remark
        for student in students:
            for comp in competencies:
                key_desc = f'item-{student.id}-{comp.id}-descriptor'
                key_score = f'item-{student.id}-{comp.id}-score'
                key_remark = f'item-{student.id}-{comp.id}-remark'
                descriptor = request.POST.get(key_desc)
                score = request.POST.get(key_score) or None
                remark = request.POST.get(key_remark) or ''
                if descriptor or score or remark:
                    ai, created = AssessmentItem.objects.update_or_create(
                        assessment=assessment, student=student, competency=comp,
                        defaults={'descriptor': descriptor, 'score': (float(score) if score else None), 'remark': remark})
        return redirect('cbc:class_dashboard', assessment.classroom.id)
    return render(request, 'cbc/competency_entry.html', {'assessment': assessment, 'students': students, 'competencies': competencies})

@login_required
def generate_report(request, student_id, term_id):
    student = get_object_or_404(Student, id=student_id)
    term = get_object_or_404(Term, id=term_id)
    return render_report_pdf_response(request, student, term)  # delegates to reports.py
    # cbc/reports.py
from django.template.loader import render_to_string
from django.http import HttpResponse
from .models import AssessmentItem
from .utils import student_area_summary
import tempfile

def render_report_pdf_response(request, student, term):
    # gather data
    items = AssessmentItem.objects.filter(student=student, assessment__term=term).select_related('competency','assessment')
    summary = student_area_summary(student, term)
    context = {'student': student, 'term': term, 'items': items, 'summary': summary, 'request': request}
    html_string = render_to_string('cbc/report_card.html', context)

    # Try WeasyPrint first
    try:
        from weasyprint import HTML
        html = HTML(string=html_string, base_url=request.build_absolute_uri('/'))
        result = html.write_pdf()
        response = HttpResponse(result, content_type='application/pdf')
        response['Content-Disposition'] = f'attachment; filename=report_{student.admission_number}_{term.name}.pdf'
        return response
    except Exception as e:
        # fallback to xhtml2pdf
        try:
            from xhtml2pdf import pisa
            response = HttpResponse(content_type='application/pdf')
            response['Content-Disposition'] = f'attachment; filename=report_{student.admission_number}_{term.name}.pdf'
            pisa_status = pisa.CreatePDF(html_string, dest=response)
            if pisa_status.err:
                return HttpResponse('We could not generate PDF at this time.')
            return response
        except Exception as e2:
            return HttpResponse('PDF generation libraries not available. Please install WeasyPrint or xhtml2pdf.')<!-- cbc/templates/cbc/base.html -->
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}CBC RMS{% endblock %}</title>
  <link href="https://cdn.jsdelivr.net/npm/tailwindcss@2/dist/tailwind.min.css" rel="stylesheet">
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body class="bg-gray-100">
  <nav class="bg-white shadow p-4">
    <div class="container mx-auto flex justify-between">
      <a class="font-bold" href="{% url 'cbc:dashboard' %}">CBC RMS</a>
      <div>
        {% if user.is_authenticated %}
          <span class="mr-4">{{ user.get_full_name }}</span>
          <a href="{% url 'logout' %}" class="text-sm text-blue-600">Logout</a>
        {% else %}
          <a href="{% url 'login' %}">Login</a>
        {% endif %}
      </div>
    </div>
  </nav>
  <div class="container mx-auto p-4">
    {% block content %}{% endblock %}
  </div>
</body>
</html><!-- cbc/templates/cbc/dashboard.html -->
{% extends "cbc/base.html" %}
{% block title %}Dashboard{% endblock %}
{% block content %}
<h1 class="text-2xl font-semibold mb-4">Dashboard</h1>
{% if profile.role == 'teacher' %}
  <h2>Your classes</h2>
  <ul>
    {% for c in classes %}
      <li><a class="text-blue-600" href="{% url 'cbc:class_dashboard' c.id %}">{{ c.name }}</a></li>
    {% empty %}
      <li>No classes assigned.</li>
    {% endfor %}
  </ul>
{% elif profile.role == 'admin' %}
  <h2>School overview</h2>
  <ul>
    {% for c in classes %}
      <li><a class="text-blue-600" href="{% url 'cbc:class_dashboard' c.id %}">{{ c.name }}</a></li>
    {% endfor %}
  </ul>
{% else %}
  <h2>Your children</h2>
  <ul>
    {% for child in children %}
      <li>{{ child.first_name }} {{ child.last_name }} — <a href="{% url 'cbc:student_reports' child.id %}">View reports</a></li>
    {% endfor %}
  </ul>
{% endif %}
{% endblock %}<!-- cbc/templates/cbc/competency_entry.html -->
{% extends "cbc/base.html" %}
{% block content %}
<h1 class="text-xl font-semibold">Enter competencies — {{ assessment.name }}</h1>
<form method="post">{% csrf_token %}
  <table class="min-w-full bg-white">
    <thead>
      <tr>
        <th>Student</th>
        {% for comp in competencies %}
          <th class="px-2 py-1">{{ comp.title }}</th>
        {% endfor %}
      </tr>
    </thead>
    <tbody>
      {% for student in students %}
        <tr class="border-t">
          <td class="p-2">{{ student.first_name }} {{ student.last_name }}</td>
          {% for comp in competencies %}
            <td class="p-1">
              <select name="item-{{ student.id }}-{{ comp.id }}-descriptor" class="w-full">
                <option value="">--</option>
                <option value="EE">EE</option>
                <option value="ME">ME</option>
                <option value="AE">AE</option>
                <option value="BE">BE</option>
              </select>
              <input name="item-{{ student.id }}-{{ comp.id }}-score" class="w-full mt-1 text-sm" placeholder="score"/>
              <textarea name="item-{{ student.id }}-{{ comp.id }}-remark" class="w-full mt-1 text-xs" placeholder="remark"></textarea>
            </td>
          {% endfor %}
        </tr>
      {% endfor %}
    </tbody>
  </table>
  <div class="mt-4">
    <button class="bg-blue-600 text-white px-4 py-2">Save</button>
  </div>
</form>
{% endblock %}<!-- cbc/templates/cbc/report_card.html -->
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <style>
    body { font-family: sans-serif; }
    .header { text-align:center; }
    .table { width:100%; border-collapse: collapse; margin-top: 12px; }
    .table th, .table td { border:1px solid #333; padding:6px; text-align:left; }
    .small { font-size: 12px; }
  </style>
</head>
<body>
  <div class="header">
    <h1>{{ student.school.name }}</h1>
    <h2>Term: {{ term.name }}</h2>
    <h3>Report Card — {{ student.first_name }} {{ student.last_name }} ({{ student.admission_number }})</h3>
  </div>

  <h4>Competency Results</h4>
  <table class="table small">
    <thead><tr><th>Subject</th><th>Strand</th><th>Sub-strand</th><th>Competency</th><th>Descriptor</th><th>Score</th><th>Remark</th></tr></thead>
    <tbody>
    {% for it in items %}
      <tr>
        <td>{{ it.competency.substrand.strand.subject.name }}</td>
        <td>{{ it.competency.substrand.strand.name }}</td>
        <td>{{ it.competency.substrand.name }}</td>
        <td>{{ it.competency.title }}</td>
        <td>{{ it.descriptor }}</td>
        <td>{{ it.score }}</td>
        <td>{{ it.remark }}</td>
      </tr>
    {% endfor %}
    </tbody>
  </table>

  <h4>Area Summaries</h4>
  <table class="table small">
    <thead><tr><th>Area</th><th>Avg Score</th><th>EE</th><th>ME</th><th>AE</th><th>BE</th></tr></thead>
    <tbody>
      {% for area, data in summary.items %}
      <tr>
        <td>{{ area }}</td>
        <td>{{ data.avg_score|floatformat:1 }}</td>
        <td>{{ data.counts.EE }}</td>
        <td>{{ data.counts.ME }}</td>
        <td>{{ data.counts.AE }}</td>
        <td>{{ data.counts.BE }}</td>
      </tr>
      {% endfor %}
    </tbody>
  </table>

  <p class="small">Generated: {{ now|default:None }}</p>
</body>
</html># cbcrms/urls.py
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('cbc.urls', namespace='cbc')),
    path('api/', include('cbc.api_urls')),  # if you create API URLs
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)# cbc/urls.py
from django.urls import path
from . import views

app_name = 'cbc'
urlpatterns = [
    path('', views.dashboard, name='dashboard'),
    path('class/<int:class_id>/', views.class_dashboard, name='class_dashboard'),
    path('class/<int:class_id>/create_assessment/', views.create_assessment, name='create_assessment'),
    path('assessment/<int:assessment_id>/entry/', views.competency_entry, name='competency_entry'),
    path('student/<int:student_id>/report/<int:term_id>/pdf/', views.generate_report, name='generate_report'),
]# cbc/views.py (add this function)
import pandas as pd
from django.contrib import messages
from django.shortcuts import render
from .models import Student

@login_required
def import_students(request):
    if request.method == 'POST' and request.FILES.get('file'):
        df = pd.read_excel(request.FILES['file'])
        created = 0
        for _, row in df.iterrows():
            Student.objects.update_or_create(
                admission_number=str(row['admission_number']).strip(),
                defaults={
                    'first_name': row.get('first_name',''),
                    'last_name': row.get('last_name',''),
                    'dob': row.get('dob', None),
                    'gender': row.get('gender',''),
                    'school': request.user.profile.school
                }
            )
            created += 1
        messages.success(request, f'{created} students processed.')
        return redirect('cbc:dashboard')
    return render(request, 'cbc/import_students.html')admission_number, first_name, last_name, dob (YYYY-MM-DD), gender# cbc/api_views.py
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from django.db.models import Avg
from .models import AssessmentItem

@api_view(['GET'])
@permission_classes([IsAuthenticated])
def class_performance(request, class_id, term_id):
    items = AssessmentItem.objects.filter(assessment__classroom_id=class_id, assessment__term_id=term_id)
    # Average per subject
    subject_avgs = items.values('competency__substrand__strand__subject__name') \
                        .annotate(avg_score=Avg('score')) \
                        .order_by('-avg_score')
    labels = [s['competency__substrand__strand__subject__name'] for s in subject_avgs]
    values = [s['avg_score'] or 0 for s in subject_avgs]
    return Response({'labels': labels, 'values': values})python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
python manage.py runserveradmission_number,first_name,last_name,dob,gender
ADM001,John,Doe,2016-05-05,M
ADM002,Jane,Doe,2016-09-10,F
django-admin startproject cbcrms
cd cbcrms
python manage.py startapp cbcpython -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
