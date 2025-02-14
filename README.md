#### Loan_system/models.py
from django.db import models

# Client (Farmer) Model
class Client(models.Model):
    first_name = models.CharField(max_length=255)
    last_name = models.CharField(max_length=255)
    phone_number = models.CharField(max_length=15)
    email = models.EmailField()
    address = models.CharField(max_length=255)
    farm_size_hectares = models.FloatField()
    registered_date = models.DateField(auto_now_add=True)

    def __str__(self):
        return f"{self.first_name} {self.last_name}"

# Loan Application Model
class Loan(models.Model):
    client = models.ForeignKey(Client, on_delete=models.CASCADE)
    loan_amount = models.DecimalField(max_digits=12, decimal_places=2)
    loan_term_months = models.PositiveIntegerField()  # Loan term in months
    interest_rate = models.DecimalField(max_digits=5, decimal_places=2)  # In percentage
    approval_date = models.DateField(auto_now_add=True)
    repayment_schedule = models.TextField()  # JSON or text describing repayment schedule

    def __str__(self):
        return f"Loan {self.id} for {self.client.first_name} {self.client.last_name}"

# Repayment Model
class Repayment(models.Model):
    loan = models.ForeignKey(Loan, on_delete=models.CASCADE)
    repayment_amount = models.DecimalField(max_digits=12, decimal_places=2)
    repayment_date = models.DateField()
    payment_method = models.CharField(max_length=50)  # E.g., MPESA, Bank Transfer, etc.
    status = models.CharField(max_length=50, choices=[('Paid', 'Paid'), ('Pending', 'Pending')])

    def __str__(self):
        return f"Repayment of {self.repayment_amount} for Loan {self.loan.id}"

        -----------------------------------
python manage.py makemigrations
python manage.py migrate
--------------------------------------
views.py
from django.shortcuts import render, redirect
from .models import Client, Loan, Repayment
from django.db.models import Sum

# View to list all clients
def client_list(request):
    clients = Client.objects.all()
    return render(request, 'loan_system/client_list.html', {'clients': clients})

# View to create a loan application
def create_loan(request, client_id):
    client = Client.objects.get(id=client_id)
    if request.method == 'POST':
        loan_amount = request.POST['loan_amount']
        loan_term_months = request.POST['loan_term_months']
        interest_rate = request.POST['interest_rate']
        loan = Loan.objects.create(
            client=client,
            loan_amount=loan_amount,
            loan_term_months=loan_term_months,
            interest_rate=interest_rate,
            repayment_schedule="Monthly repayment plan (details to be added)",
        )
        return redirect('loan_detail', loan_id=loan.id)
    return render(request, 'loan_system/create_loan.html', {'client': client})

# View to list loans and repayment status
def loan_list(request):
    loans = Loan.objects.all()
    return render(request, 'loan_system/loan_list.html', {'loans': loans})

# View to list repayments for a specific loan
def repayment_list(request, loan_id):
    loan = Loan.objects.get(id=loan_id)
    repayments = Repayment.objects.filter(loan=loan)
    total_repaid = repayments.aggregate(Sum('repayment_amount'))['repayment_amount__sum'] or 0
    return render(request, 'loan_system/repayment_list.html', {'loan': loan, 'repayments': repayments, 'total_repaid': total_repaid})
--------------------------------------
### Client List
## client_list.html
<!DOCTYPE html>
<html>
<head>
    <title>Farm Clients</title>
</head>
<body>
    <h1>Clients</h1>
    <table>
        <tr>
            <th>First Name</th>
            <th>Last Name</th>
            <th>Farm Size (Hectares)</th>
            <th>Loan Application</th>
        </tr>
        {% for client in clients %}
        <tr>
            <td>{{ client.first_name }}</td>
            <td>{{ client.last_name }}</td>
            <td>{{ client.farm_size_hectares }}</td>
            <td><a href="{% url 'create_loan' client.id %}">Apply for Loan</a></td>
        </tr>
        {% endfor %}
    </table>
</body>
</html>
-------------------------------------
## Loan_list.html
<!DOCTYPE html>
<html>
<head>
    <title>Loans</title>
</head>
<body>
    <h1>Loans</h1>
    <table>
        <tr>
            <th>Client Name</th>
            <th>Loan Amount</th>
            <th>Term (Months)</th>
            <th>Repayment Status</th>
        </tr>
        {% for loan in loans %}
        <tr>
            <td>{{ loan.client.first_name }} {{ loan.client.last_name }}</td>
            <td>{{ loan.loan_amount }}</td>
            <td>{{ loan.loan_term_months }}</td>
            <td><a href="{% url 'repayment_list' loan.id %}">View Repayments</a></td>
        </tr>
        {% endfor %}
    </table>
</body>
</html>
------------------------------------
## create_loan.html
<!DOCTYPE html>
<html>
<head>
    <title>Create Loan</title>
</head>
<body>
    <h1>Apply for Loan - {{ client.first_name }} {{ client.last_name }}</h1>
    <form method="POST">
        {% csrf_token %}
        <label for="loan_amount">Loan Amount:</label>
        <input type="text" name="loan_amount" required><br><br>
        <label for="loan_term_months">Loan Term (Months):</label>
        <input type="text" name="loan_term_months" required><br><br>
        <label for="interest_rate">Interest Rate (%):</label>
        <input type="text" name="interest_rate" required><br><br>
        <button type="submit">Submit</button>
    </form>
</body>
</html>
----------------------------------------
## repayment_list.html
<!DOCTYPE html>
<html>
<head>
    <title>Repayments for Loan {{ loan.id }}</title>
</head>
<body>
    <h1>Repayments for Loan {{ loan.id }}</h1>
    <table>
        <tr>
            <th>Repayment Amount</th>
            <th>Repayment Date</th>
            <th>Status</th>
        </tr>
        {% for repayment in repayments %}
        <tr>
            <td>{{ repayment.repayment_amount }}</td>
            <td>{{ repayment.repayment_date }}</td>
            <td>{{ repayment.status }}</td>
        </tr>
        {% endfor %}
    </table>
    <h3>Total Repaid: {{ total_repaid }}</h3>
</body>
</html>
-----------------------------------------------
## farm_loan_management/urls.py
from django.urls import path
from loan_system import views

urlpatterns = [
    path('clients/', views.client_list, name='client_list'),
    path('loan/<int:client_id>/create/', views.create_loan, name='create_loan'),
    path('loans/', views.loan_list, name='loan_list'),
    path('loan/<int:loan_id>/repayments/', views.repayment_list, name='repayment_list'),
]
--------------------------------------------------
### loan_system/admin.py
from django.contrib import admin
from .models import Client, Loan, Repayment

admin.site.register(Client)
admin.site.register(Loan)
admin.site.register(Repayment)
-----------------------------------------------------
### Dashboard Update
#### Views.py
from django.shortcuts import render
from django.db.models import Sum
from .models import Loan, Repayment

# Dashboard View to show loan stats
def dashboard(request):
    # Total number of loans
    total_loans = Loan.objects.count()

    # Total amount of loans disbursed
    total_loan_amount = Loan.objects.aggregate(Sum('loan_amount'))['loan_amount__sum'] or 0

    # Total repayments made
    total_repayments = Repayment.objects.aggregate(Sum('repayment_amount'))['repayment_amount__sum'] or 0

    # Repayment rate (percentage of total repayments vs loan amount)
    repayment_rate = (total_repayments / total_loan_amount) * 100 if total_loan_amount else 0

    # Get the outstanding balance for each client
    outstanding_balances = []
    loans = Loan.objects.all()
    for loan in loans:
        repayments = Repayment.objects.filter(loan=loan)
        total_repaid = repayments.aggregate(Sum('repayment_amount'))['repayment_amount__sum'] or 0
        outstanding_balance = loan.loan_amount - total_repaid
        outstanding_balances.append({
            'client': loan.client,
            'outstanding_balance': outstanding_balance,
            'loan': loan,
            'repayment_schedule': loan.repayment_schedule
        })

    context = {
        'total_loans': total_loans,
        'repayment_rate': repayment_rate,
        'outstanding_balances': outstanding_balances,
    }

    return render(request, 'loan_system/dashboard.html', context)

Step 2: Create the Dashboard Template
We will create a template to show the dashboard data, including:

Number of Loans
Repayment Rate
Outstanding Balance for each Loan
Repayment Plan for each Client
2.1. Dashboard Template (dashboard.html)
html
Copy
Edit
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Loan Dashboard</title>
    <style>
        /* Basic Styling */
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
        }

        .container {
            width: 80%;
            margin: 20px auto;
        }

        .dashboard-stats {
            display: flex;
            justify-content: space-around;
            margin-bottom: 20px;
        }

        .stat-box {
            padding: 20px;
            border: 1px solid #ccc;
            border-radius: 5px;
            text-align: center;
            width: 30%;
        }

        .stat-box h2 {
            margin-bottom: 10px;
            font-size: 24px;
        }

        .stat-box p {
            font-size: 18px;
        }

        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 30px;
        }

        table, th, td {
            border: 1px solid #ccc;
        }

        th, td {
            padding: 12px;
            text-align: center;
        }

        th {
            background-color: #f4f4f4;
        }

        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            cursor: pointer;
            border-radius: 5px;
        }

        .btn:hover {
            background-color: #45a049;
        }
    </style>
</head>
<body>

<div class="container">
    <!-- Dashboard Stats -->
    <div class="dashboard-stats">
        <div class="stat-box">
            <h2>Total Loans</h2>
            <p>{{ total_loans }}</p>
        </div>
        <div class="stat-box">
            <h2>Repayment Rate (%)</h2>
            <p>{{ repayment_rate|floatformat:2 }}%</p>
        </div>
        <div class="stat-box">
            <h2>Total Loan Amount</h2>
            <p>{{ total_loan_amount }}</p>
        </div>
    </div>

    <!-- Outstanding Balances Table -->
    <h2>Outstanding Balances and Repayment Plans</h2>
    <table>
        <tr>
            <th>Client Name</th>
            <th>Loan Amount</th>
            <th>Outstanding Balance</th>
            <th>Repayment Schedule</th>
        </tr>
        {% for item in outstanding_balances %}
        <tr>
            <td>{{ item.client.first_name }} {{ item.client.last_name }}</td>
            <td>{{ item.loan.loan_amount }}</td>
            <td>{{ item.outstanding_balance }}</td>
            <td>{{ item.repayment_schedule }}</td>
        </tr>
        {% endfor %}
    </table>

    <br>
    <a href="/admin/" class="btn">Go to Admin Panel</a>
</div>

</body>
</html>


Step 3: URL Configuration
Update your URLs to include the dashboard view.

In farm_loan_management/urls.py, add the dashboard route:

python
Copy
Edit
from django.urls import path
from loan_system import views

urlpatterns = [
    path('dashboard/', views.dashboard, name='dashboard'),
    path('clients/', views.client_list, name='client_list'),
    path('loan/<int:client_id>/create/', views.create_loan, name='create_loan'),
    path('loans/', views.loan_list, name='loan_list'),
    path('loan/<int:loan_id>/repayments/', views.repayment_list, name='repayment_list'),
]


