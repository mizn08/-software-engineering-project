# -software-engineering-project
 DevOps Project Assessment Document
<?php
session_start();
require_once '../support/db.php';

if (!isset($_SESSION['email'])) {
    header("Location: ../login.php");
    exit();
}

// Step 1: Get the officer's municipality
$email = $_SESSION['email']; // Email from session

$officerQuery = "SELECT municipal FROM officer WHERE email = ?";
$stmt = $conn->prepare($officerQuery);
$stmt->bind_param("s", $email);
$stmt->execute();
$result = $stmt->get_result();

if ($result->num_rows > 0) {
    $row = $result->fetch_assoc();
    $municipality = $row['municipal'];
} else {
    die("Municipality not found for this officer.");
}

// Step 2: Fetch complaint data based on municipality
$sql = "SELECT status, COUNT(*) AS count 
        FROM complaints 
        WHERE municipality = ? 
        GROUP BY status";

$stmt = $conn->prepare($sql);
$stmt->bind_param("s", $municipality);
$stmt->execute();
$result = $stmt->get_result();

$open_tickets = $pending_tickets = $close_tickets = 0;

if ($result->num_rows > 0) {
    while ($row = $result->fetch_assoc()) {
        switch ($row['status']) {
            case 'open':
                $open_tickets = $row['count'];
                break;
            case 'pending':
                $pending_tickets = $row['count'];
                break;
            case 'close':
                $close_tickets = $row['count'];
                break;
        }
    }
}

$stmt->close();
$conn->close();

$total_tickets = $open_tickets + $pending_tickets + $close_tickets;
?>

<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>City Officer Dashboard</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet" />
    <style>
        /* Body styling */
        body {
            font-family: 'Arial', sans-serif;
            background: url('city_pic.jpg') no-repeat center center fixed;
            background-size: cover;
            color: #fff;
            margin: 0;
            padding: 0;
            min-height: 100vh;
        }

       /* Semi-transparent overlay for contrast */
        body::before {
            content: '';
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            background: rgba(0, 0, 0, 0.5); /* Dark overlay for better readability */
            z-index: 0;
        }

        .main-content {
            margin-left: 250px;
            padding: 25px;
            position: relative;
            z-index: 1; /* Place content above the overlay */
        }

        h2 {
            text-align: center;
            margin-bottom: 20px;
            font-weight: 600;
            font-size: 32px;
            color: #EDF7F6; /* Vibrant color for young appeal */
        }

        /* Status Cards Styling */
        .status-cards {
            display: flex;
            flex-wrap: wrap;
            justify-content: space-evenly;
            gap: 20px;
        }

        .card {
            background: linear-gradient(135deg, #2d545e, #e1b382);
            color: #fff;
            border: none;
            border-radius: 15px;
            box-shadow: 0px 5px 15px rgba(0, 0, 0, 0.2);
            width: 250px;
            padding: 20px;
            text-align: center;
            transition: transform 0.3s ease, box-shadow 0.3s ease;
        }

        .card:hover {
            transform: translateY(-8px);
            box-shadow: 0px 8px 20px rgba(0, 0, 0, 0.3);
        }

        .card-title {
            font-size: 18px;
            font-weight: 600;
        }

        .card-text {
            font-size: 24px;
            font-weight: bold;
        }

        /* Chart Container */
        .chart-container {
            background: linear-gradient(135deg, #EDF7F6, #A28089);
            border-radius: 15px;
            padding: 20px;
            width: 100%;
            max-width: 400px;
            margin: 20px auto;
            box-shadow: 0px 5px 15px rgba(0, 0, 0, 0.2);
        }

        .chart-container canvas {
            max-width: 100%;
        }

        /* Responsive Design */
        @media (max-width: 768px) {
            .main-content {
                margin-left: 0;
                padding: 15px;
            }

            h2 {
                font-size: 28px;
            }

            .status-cards {
                flex-direction: column;
                align-items: center;
            }

            .card {
                width: 90%;
            }

            .chart-container {
                width: 90%;
            }
        }
    </style>
</head>

<body>
    <!-- Include Header -->
    <?php include 'head.php'; ?>

    <!-- Include Sidebar -->
    <?php include 'sidebar.php'; ?>

    <!-- Main Content -->
    <div class="main-content">
        <h2>City Officer Dashboard</h2>
        <div class="status-cards">
            <div class="card">
                <h5 class="card-title">Total Tickets</h5>
                <p class="card-text"><?php echo $total_tickets; ?></p>
            </div>
            <div class="card">
                <h5 class="card-title">Open Tickets</h5>
                <p class="card-text"><?php echo $open_tickets; ?></p>
            </div>
            <div class="card">
                <h5 class="card-title">Pending Tickets</h5>
                <p class="card-text"><?php echo $pending_tickets; ?></p>
            </div>
            <div class="card">
                <h5 class="card-title">Closed Tickets</h5>
                <p class="card-text"><?php echo $close_tickets; ?></p>
            </div>
        </div>

        <div class="chart-container">
            <canvas id="ticketStatusChart"></canvas>
        </div>
    </div>

    <!-- Scripts -->
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script>
        const ctx = document.getElementById('ticketStatusChart').getContext('2d');
        const ticketStatusChart = new Chart(ctx, {
            type: 'doughnut',
            data: {
                labels: ['Open', 'Pending', 'Closed'],
                datasets: [{
                    label: 'Ticket Status',
                    data: [
                        <?php echo $open_tickets; ?>,
                        <?php echo $pending_tickets; ?>,
                        <?php echo $close_tickets; ?>
                    ],
                    backgroundColor: ['#42A5F5', '#FFA726', '#66BB6A'],
                    hoverOffset: 6
                }]
            },
            options: {
                responsive: true,
                plugins: {
                    legend: {
                        position: 'top',
                        labels: {
                            color: '#fff',
                            font: {
                                size: 14
                            }
                        }
                    },
                    tooltip: {
                        callbacks: {
                            label: function (context) {
                                let label = context.label || '';
                                if (label) {
                                    label += ': ';
                                }
                                label += context.raw;
                                return label;
                            }
                        }
                    }
                }
            }
        });
    </script>
</body>

</html>
