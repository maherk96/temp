grep -Poz "(?s)8=FIX.*?35=D.*?(?=8=FIX|$)" your_log_file.log > new_orders.log
