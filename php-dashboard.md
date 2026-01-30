<?php
// config.php
define('DB_PATH', __DIR__ . '/content.db');
define('ANALYTICS_CACHE_TIME', 3600); // 1 hour

class ContentDashboard {
    private $db;
    
    public function __construct() {
        $this->db = new SQLite3(DB_PATH);
    }
    
    public function getRecentPosts($limit = 10, $platform = null) {
        $query = "SELECT * FROM posts ";
        if ($platform) {
            $query .= "WHERE platform = :platform ";
        }
        $query .= "ORDER BY created_at DESC LIMIT :limit";
        
        $stmt = $this->db->prepare($query);
        $stmt->bindValue(':limit', $limit, SQLITE3_INTEGER);
        if ($platform) {
            $stmt->bindValue(':platform', $platform, SQLITE3_TEXT);
        }
        
        return $this->fetchAll($stmt->execute());
    }
    
    public function getContentAnalytics($days = 7) {
        $query = "
            SELECT 
                platform,
                COUNT(*) as post_count,
                AVG(json_extract(metrics, '$.engagement')) as avg_engagement,
                MAX(json_extract(metrics, '$.engagement')) as max_engagement
            FROM posts 
            WHERE created_at >= datetime('now', '-' || :days || ' days')
            GROUP BY platform
        ";
        
        $stmt = $this->db->prepare($query);
        $stmt->bindValue(':days', $days, SQLITE3_INTEGER);
        
        return $this->fetchAll($stmt->execute());
    }
    
    public function getGeneratedContent($post_id) {
        $query = "
            SELECT * FROM generated_content 
            WHERE original_post_id = :post_id 
            ORDER BY created_at DESC
        ";
        
        $stmt = $this->db->prepare($query);
        $stmt->bindValue(':post_id', $post_id, SQLITE3_INTEGER);
        
        return $this->fetchAll($stmt->execute());
    }
    
    private function fetchAll($result) {
        $rows = [];
        while ($row = $result->fetchArray(SQLITE3_ASSOC)) {
            $rows[] = $row;
        }
        return $rows;
    }
    
    public function getContentCalendar($start_date, $end_date) {
        $query = "
            SELECT 
                posts.*,
                generated_content.content as generated_variation
            FROM posts 
            LEFT JOIN generated_content ON posts.id = generated_content.original_post_id
            WHERE posts.publish_date BETWEEN :start_date AND :end_date
            ORDER BY posts.publish_date ASC
        ";
        
        $stmt = $this->db->prepare($query);
        $stmt->bindValue(':start_date', $start_date, SQLITE3_TEXT);
        $stmt->bindValue(':end_date', $end_date, SQLITE3_TEXT);
        
        return $this->fetchAll($stmt->execute());
    }
}

// API endpoint for AJAX requests
if (isset($_GET['action'])) {
    header('Content-Type: application/json');
    $dashboard = new ContentDashboard();
    
    switch ($_GET['action']) {
        case 'recent_posts':
            echo json_encode($dashboard->getRecentPosts(
                $_GET['limit'] ?? 10,
                $_GET['platform'] ?? null
            ));
            break;
            
        case 'analytics':
            echo json_encode($dashboard->getContentAnalytics(
                $_GET['days'] ?? 7
            ));
            break;
            
        case 'calendar':
            echo json_encode($dashboard->getContentCalendar(
                $_GET['start_date'],
                $_GET['end_date']
            ));
            break;
    }
    exit;
}
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Content Analytics Dashboard</title>
    <link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body class="bg-gray-100">
    <div class="container mx-auto px-4 py-8">
        <header class="mb-8">
            <h1 class="text-3xl font-bold text-gray-800">Content Analytics Dashboard</h1>
        </header>
        
        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
            <!-- Analytics Overview -->
            <div class="bg-white rounded-lg shadow p-6">
                <h2 class="text-xl font-semibold mb-4">Performance Overview</h2>
                <canvas id="analyticsChart"></canvas>
            </div>
            
            <!-- Recent Posts -->
            <div class="bg-white rounded-lg shadow p-6">
                <h2 class="text-xl font-semibold mb-4">Recent Posts</h2>
                <div id="recentPosts" class="space-y-4"></div>
            </div>
            
            <!-- Content Calendar -->
            <div class="bg-white rounded-lg shadow p-6">
                <h2 class="text-xl font-semibold mb-4">Content Calendar</h2>
                <div id="contentCalendar"></div>
            </div>
        </div>
    </div>

    <script>
        // Dashboard JavaScript implementation
        async function loadDashboard() {
            // Load analytics data
            const analyticsData = await fetch('?action=analytics').then(r => r.json());
            
            // Create analytics chart
            const ctx = document.getElementById('analyticsChart').getContext('2d');
            new Chart(ctx, {
                type: 'bar',
                data: {
                    labels: analyticsData.map(d => d.platform),
                    datasets: [{
                        label: 'Average Engagement',
                        data: analyticsData.map(d => d.avg_engagement)
                    }]
                }
            });
            
            // Load recent posts
            const posts = await fetch('?action=recent_posts').then(r => r.json());
            document.getElementById('recentPosts').innerHTML = posts
                .map(post => `
                    <div class="border-b pb-4">
                        <div class="font-medium">${post.content}</div>
                        <div class="text-sm text-gray-500">
                            ${post.platform} • ${new Date(post.created_at).toLocaleDateString()}
                        </div>
                    </div>
                `)
                .join('');
        }

        document.addEventListener('DOMContentLoaded', loadDashboard);
    </script>
</body>
</html>