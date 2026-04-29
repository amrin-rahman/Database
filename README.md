# Database
```java

import javax.swing.*;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.sql.*;
import java.util.ArrayList;

public class Db_project1 extends JFrame {

    Connection con;
    int userId = -1;

    JTable productTable, orderTable;
    DefaultTableModel productModel, orderModel;

    ArrayList<Integer> cart = new ArrayList<>();

    public Db_project1() {
        connect();
        loginUI();
    }

   
    void connect() {
        try {
            Class.forName("com.mysql.cj.jdbc.Driver");

     con = DriverManager.getConnection(
        "jdbc:mysql://localhost:3306/db_project1?useSSL=false&serverTimezone=UTC",
        "root",
        ""
);

            System.out.println("DB Connected Successfully!");

        } catch (Exception e) {
            JOptionPane.showMessageDialog(this, "Database Connection Failed!");
            e.printStackTrace();
        }
    }


    void loginUI() {
        JFrame f = new JFrame("Login");
        f.setSize(300, 200);
        f.setLayout(new GridLayout(3, 2));

        JTextField email = new JTextField();
        JPasswordField pass = new JPasswordField();

        f.add(new JLabel("Email"));
        f.add(email);
        f.add(new JLabel("Password"));
        f.add(pass);

        JButton login = new JButton("Login");
        f.add(login);

        login.addActionListener(e -> {
            try {
                PreparedStatement ps = con.prepareStatement(
                        "SELECT * FROM users WHERE email=? AND password=?"
                );

                ps.setString(1, email.getText());
                ps.setString(2, new String(pass.getPassword()));

                ResultSet rs = ps.executeQuery();

                if (rs.next()) {
                    userId = rs.getInt("user_id");
                    f.dispose();
                    mainUI();
                } else {
                    JOptionPane.showMessageDialog(f, "Invalid Login");
                }

            } catch (Exception ex) {
                ex.printStackTrace();
            }
        });

        f.setVisible(true);
    }


    void mainUI() {
        setTitle("E-Commerce System (DBPROJECT)");
        setSize(1000, 600);
        setLayout(new BorderLayout());

        productModel = new DefaultTableModel();
        productModel.setColumnIdentifiers(new String[]{"ID", "Name", "Price", "Stock", "Status"});

        productTable = new JTable(productModel);
        loadProducts();

        orderModel = new DefaultTableModel();
        orderModel.setColumnIdentifiers(new String[]{"Order ID", "Total", "Status"});

        orderTable = new JTable(orderModel);
        loadOrders();

        JSplitPane split = new JSplitPane(
                JSplitPane.VERTICAL_SPLIT,
                new JScrollPane(productTable),
                new JScrollPane(orderTable)
        );

        add(split, BorderLayout.CENTER);

        JPanel panel = new JPanel();

        JButton addCart = new JButton("Add Cart");
        JButton orderBtn = new JButton("Place Order");
        JButton wishlistBtn = new JButton("Wishlist");
        JButton cancelBtn = new JButton("Cancel Order");
        JButton trackBtn = new JButton("Track");

        panel.add(addCart);
        panel.add(orderBtn);
        panel.add(wishlistBtn);
        panel.add(cancelBtn);
        panel.add(trackBtn);

        add(panel, BorderLayout.SOUTH);

        addCart.addActionListener(e -> addToCart());
        orderBtn.addActionListener(e -> placeOrder());
        wishlistBtn.addActionListener(e -> addWishlist());
        cancelBtn.addActionListener(e -> cancelOrder());
        trackBtn.addActionListener(e -> trackOrder());

        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setVisible(true);
    }

   
    void loadProducts() {
        try {
            productModel.setRowCount(0);

            ResultSet rs = con.createStatement()
                    .executeQuery("SELECT * FROM products");

            while (rs.next()) {
                productModel.addRow(new Object[]{
                        rs.getInt("product_id"),
                        rs.getString("name"),
                        rs.getDouble("price"),
                        rs.getInt("stock"),
                        rs.getString("status")
                });
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

 
    void loadOrders() {
        try {
            orderModel.setRowCount(0);

            PreparedStatement ps = con.prepareStatement(
                    "SELECT * FROM orders WHERE user_id=?"
            );

            ps.setInt(1, userId);

            ResultSet rs = ps.executeQuery();

            while (rs.next()) {
                orderModel.addRow(new Object[]{
                        rs.getInt("order_id"),
                        rs.getDouble("total"),
                        rs.getString("status")
                });
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }


    void addToCart() {
        int row = productTable.getSelectedRow();
        if (row == -1) return;

        int id = (int) productModel.getValueAt(row, 0);
        cart.add(id);

        JOptionPane.showMessageDialog(this, "Added to Cart");
    }


    void placeOrder() {
        try {
            double total = 0;

            for (int id : cart) {
                PreparedStatement ps = con.prepareStatement(
                        "SELECT price, stock FROM products WHERE product_id=?"
                );

                ps.setInt(1, id);
                ResultSet rs = ps.executeQuery();

                if (rs.next()) {
                    total += rs.getDouble("price");

                    con.createStatement().executeUpdate(
                            "UPDATE products SET stock = stock - 1 WHERE product_id=" + id
                    );
                }
            }

            PreparedStatement ps = con.prepareStatement(
                    "INSERT INTO orders(user_id,total,status) VALUES(?,?,?)",
                    Statement.RETURN_GENERATED_KEYS
            );

            ps.setInt(1, userId);
            ps.setDouble(2, total);
            ps.setString(3, "PLACED");

            ps.executeUpdate();

            ResultSet rs = ps.getGeneratedKeys();
            int orderId = 0;
            if (rs.next()) orderId = rs.getInt(1);

            con.createStatement().executeUpdate(
                    "INSERT INTO order_tracking(order_id,status) VALUES(" + orderId + ",'PLACED')"
            );

            cart.clear();
            loadOrders();
            loadProducts();

            JOptionPane.showMessageDialog(this, "Order Placed! Total = " + total);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

   
    void addWishlist() {
        int row = productTable.getSelectedRow();
        if (row == -1) return;

        int pid = (int) productModel.getValueAt(row, 0);

        try {
            PreparedStatement ps = con.prepareStatement(
                    "INSERT INTO wishlist(user_id,product_id) VALUES(?,?)"
            );

            ps.setInt(1, userId);
            ps.setInt(2, pid);
            ps.executeUpdate();

            JOptionPane.showMessageDialog(this, "Added to Wishlist");

        } catch (Exception e) {
            e.printStackTrace();
        }
    }


    void cancelOrder() {
        int row = orderTable.getSelectedRow();
        if (row == -1) return;

        int orderId = (int) orderModel.getValueAt(row, 0);

        try {
            con.createStatement().executeUpdate(
                    "UPDATE orders SET status='CANCELLED' WHERE order_id=" + orderId
            );

            loadOrders();
            JOptionPane.showMessageDialog(this, "Order Cancelled");

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

   
    void trackOrder() {
        int row = orderTable.getSelectedRow();
        if (row == -1) return;

        int orderId = (int) orderModel.getValueAt(row, 0);

        try {
            ResultSet rs = con.createStatement().executeQuery(
                    "SELECT status FROM order_tracking WHERE order_id=" + orderId
            );

            StringBuilder sb = new StringBuilder();

            while (rs.next()) {
                sb.append(rs.getString(1)).append("\n");
            }

            JOptionPane.showMessageDialog(this, sb.toString());

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        new Db_project1();
    }
}
```
