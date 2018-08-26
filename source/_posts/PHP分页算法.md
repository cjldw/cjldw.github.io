---
title: PHP分页算法
tags: []
date: 2015-08-27 22:32:00
---

简单php分页算法

<!-- more -->

## PHP分页算法 ##

```html
    <html>
        <head>
            <link rel="stylesheet" href="//cdn.bootcss.com/bootstrap/3.3.5/css/bootstrap.min.css">
            <link rel="stylesheet" href="//cdn.bootcss.com/bootstrap/3.3.5/css/bootstrap-theme.min.css">
            <script src="//cdn.bootcss.com/jquery/1.11.3/jquery.min.js"></script>
            <script src="//cdn.bootcss.com/bootstrap/3.3.5/js/bootstrap.min.js"></script>
        </head>
    <body>
        <div class="container">
    <?php
    $page = isset($_GET['page']) ? $_GET['page'] : 1;
    echo getPaginationString($page, $total = 80, $limit = 5, $adjacents = 1, $targetpage = '/php/demo.php', $pagestring = '?page=');

    //$page = isset($_GET['page']) ? $_GET['page'] : 1;
    //echo getPaginationString2($page, $total = 80, $limit = 5, $showPageCount = 5, $targetpage = '/php/demo.php', $pagestring = '?page=');
    ?>
    </div>
    </body>
    </html>
    <?php
    //function to return the pagination string
    function getPaginationString($page = 1, $totalitems, $limit = 15, $adjacents = 3, $targetpage = "/", $pagestring = "?page=")
    {
        //defaults
        if(!$adjacents) $adjacents = 1;
        if(!$limit) $limit = 15;
        if(!$page) $page = 1;
        if(!$targetpage) $targetpage = "/";

        //other vars
        $prev = $page - 1;                                    //previous page is page - 1
        $next = $page + 1;                                    //next page is page + 1
        $lastpage = ceil($totalitems / $limit);                //lastpage is = total items / items per page, rounded up.
        $lpm1 = $lastpage - 1;                                //last page minus 1

        /*
            Now we apply our rules and draw the pagination object.
            We're actually saving the code to a variable in case we want to draw it more than once.
        */
        $pagination = "";
        if($lastpage > 1)
        {
            $pagination .= "

    ";

            //previous button
            if ($page > 1)
                $pagination .= "*   [« prev](\)
    ";
            else
                $pagination .= "*   <span class=\"disabled\">« prev</span>
    ";

            //pages
            if ($lastpage < 7 + ($adjacents * 2))    //not enough pages to bother breaking it up
            {
                for ($counter = 1; $counter <= $lastpage; $counter++)
                {
                    if ($counter == $page)
                        $pagination .= "*   <span class=\"current\">$counter</span>
    ";
                    else
                        $pagination .= "*   [$counter](\)
    ";
                }
            }
            elseif($lastpage >= 7 + ($adjacents * 2))    //enough pages to hide some
            {
                //close to beginning; only hide later pages
                if($page < 1 + ($adjacents * 3))
                {
                    for ($counter = 1; $counter < 4 + ($adjacents * 2); $counter++)
                    {
                        if ($counter == $page)
                            $pagination .= "*   <span class=\"current\">$counter</span>
    ";
                        else
                            $pagination .= "*   [$counter](\)
    ";
                    }
                    $pagination .= "*   <span class=\"elipses\">...</span>
    ";
                    $pagination .= "*   [$lpm1](\)
    ";
                    $pagination .= "*   [$lastpage](\)
    ";
                }
                //in middle; hide some front and some back
                elseif($lastpage - ($adjacents * 2) > $page && $page > ($adjacents * 2))
                {
                    $pagination .= "*   [1](\)
    ";
                    $pagination .= "*   [2](\)
    ";
                    $pagination .= "*   <span class=\"elipses\">...</span>
    ";
                    for ($counter = $page - $adjacents; $counter <= $page + $adjacents; $counter++)
                    {
                        if ($counter == $page)
                            $pagination .= "*   <span class=\"current\">$counter</span>
    ";
                        else
                            $pagination .= "*   [$counter](\)
    ";
                    }
                    $pagination .= "*   [...](javascript:;)
    ";
                    $pagination .= "*   [$lpm1](\)
    ";
                    $pagination .= "*   [$lastpage](\)
    ";
                }
                //close to end; only hide early pages
                else
                {
                    $pagination .= "*   [1](\)
    ";
                    $pagination .= "*   [2](\)
    ";
                    $pagination .= "*   <span class=\"elipses\">...</span>
    ";
                    for ($counter = $lastpage - (1 + ($adjacents * 3)); $counter <= $lastpage; $counter++)
                    {
                        if ($counter == $page)
                            $pagination .= "*   <span class=\"current\">$counter</span>
    ";
                        else
                            $pagination .= "*   [$counter](\)
    ";
                    }
                }
            }

            //next button
            if ($page < $counter - 1)
                $pagination .= "*   [next »](\)
    ";
            else
                $pagination .= "*   <span class=\"disabled\">next &raquo;</span>\n";
        }

        return $pagination;

    }

    /**
    function getPaginationString2($page = null, $totalItems = null, $limit = 5, $showPageCount = 4, $target = '/', $pageString = '?page=')
    {
        $totalPage = ceil($totalItems / $limit); // compute total page

        if(($offset = ($page + floor($showPageCount / 2))) > $totalPage)
            $offset = $totalPage;

        if($page + floor($showPageCount / 2) > $totalPage)
            $counter = $totalPage - floor($showPageCount / 2);
        else
            $counter = ($computerPage = $page - floor($showPageCount / 2)) > 0 ? $computerPage : 1;

        $pagination = '

    ';
        for($counter; $counter < $offset; $counter++)
        {
            $pagination .= '*   ['.$counter . ']()';

        return $pagination;
    }
     */
```

![](http://images0.cnblogs.com/blog2015/451327/201508/272231215945352.png)
