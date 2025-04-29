# expens-treacer
##ExpenseControllerTest.java


import com.example.controller.ExpenseController;
import com.example.model.Expense;
import com.example.model.ExpenseManager;
import com.example.service.ExpenseService;
import com.example.view.GUI;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.MockedStatic;
import org.mockito.Mockito;

import javax.swing.*;
import javax.swing.table.DefaultTableModel;
import java.io.IOException;
import java.time.YearMonth;
import java.util.ArrayList;
import java.util.List;
import java.util.TreeMap;

import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class ExpenseControllerTest {

    private ExpenseController controller;
    private ExpenseManager mockManager;
    private GUI mockGui;
    private ExpenseService mockService;

    @BeforeEach
    void setUp() {
        mockService = mock(ExpenseService.class);
        mockManager = mock(ExpenseManager.class);
        mockGui = mock(GUI.class);
        controller = new ExpenseController(mockManager);
        controller.setGui(mockGui);
        mockGui.tableModel = new DefaultTableModel();
        mockGui.expensesTable = new JTable(mockGui.tableModel);
        when(mockManager.getService()).thenReturn(mockService);

        // Initialize monthComboBox with YearMonth items or Strings, depending on your application's needs
        JComboBox<YearMonth> monthComboBox = new JComboBox<>();
        monthComboBox.addItem(YearMonth.now()); // Add the current month as an item
        // Ensure there's a selected item
        monthComboBox.setSelectedIndex(0);

        // Assuming your GUI class has a public or package-private monthComboBox field
        mockGui.monthComboBox = monthComboBox;
    }

    @Test
    void testAddExpenseSuccessfully() {
        when(mockManager.addExpense(any(Expense.class))).thenReturn(true);
        boolean result = controller.addExpense("Lunch", "01/04/2023", "Food", "10.5", "USD");
        verify(mockManager, times(1)).addExpense(any(Expense.class));
        verify(mockGui, times(1)).updateMonthComboBox();
        assertTrue(result);
    }


    @Test
    void testAddExpenseWithInvalidAmount() {
        boolean result = controller.addExpense("Dinner", "01/04/2023", "Food", "notANumber", "USD");
        verify(mockGui, never()).updateMonthComboBox();
        assertFalse(result);
    }

    @Test
    void testRemoveSelectedExpense() {
        controller.addExpense("Test Expense", "07/09/1981", "Food", "0.0", "USD");
        mockGui.expensesTable.addRowSelectionInterval(0, 0);
        when(mockManager.removeExpense(anyInt())).thenReturn(true);

        try (MockedStatic<JOptionPane> mockedStatic = Mockito.mockStatic(JOptionPane.class)) {
            mockedStatic.when(() -> JOptionPane.showConfirmDialog(
                            any(), anyString(), anyString(), anyInt(), anyInt()))
                    .thenReturn(JOptionPane.YES_OPTION);

            controller.removeSelectedExpense();

            verify(mockManager, times(1)).removeExpense(anyInt());
            verify(mockGui, times(2)).updateMonthComboBox();
        }
    }

    @Test
    void testEditSelectedExpense() throws IOException {
        // Arrange
        mockGui.expensesTable = mock(JTable.class);
        mockGui.monthComboBox = mock(JComboBox.class);

        // Mock GUI interactions
        when(mockGui.expensesTable.getSelectedRow()).thenReturn(0);
        when(mockGui.monthComboBox.getSelectedItem()).thenReturn(YearMonth.now());
        doNothing().when(mockGui).updateMonthComboBox();
        doNothing().when(mockGui).updateTableRow(any(Expense.class), anyInt());
        doNothing().when(mockGui).updateTotalExpensesByCategoryDisplay();

        // Prepare the data
        List<Expense> expenses = new ArrayList<>();
        Expense oldExpense = new Expense("Test Expense", "07/09/1981", "Food", 0.0, "USD");
        expenses.add(oldExpense);
        when(mockManager.getAllExpenses()).thenReturn(expenses);
        when(mockManager.getExpensesGroupedByMonth()).thenReturn(new TreeMap<YearMonth, List<Expense>>() {{
            put(YearMonth.now(), expenses);
        }});

        // Simulate user input for editing an expense
        Expense newExpense = new Expense("Edited Expense", "07/09/1981", "Food", 20.0, "USD");
        when(mockManager.getService()).thenReturn(mockService);
        doNothing().when(mockService).convertExpenseCurrency(any(Expense.class), anyString());

        // Mock the getUpdatedExpenseFromUser method to avoid UI interaction
        ExpenseController controllerSpy = spy(controller);
        doReturn(newExpense).when(controllerSpy).getUpdatedExpenseFromUser(oldExpense);
        when(mockManager.editExpense(any(Expense.class), any(Expense.class))).thenReturn(true);

        // Act
        controllerSpy.editSelectedExpense();

        // Assert
        verify(mockManager, times(1)).editExpense(any(Expense.class), any(Expense.class));
        verify(mockGui, times(1)).updateMonthComboBox();
        verify(mockGui, times(1)).updateTableRow(any(Expense.class), anyInt());
        if (mockGui.totalsPanelVisible) {
            verify(mockGui, times(1)).updateTotalExpensesByCategoryDisplay();
        }
    }


    @Test
    void testConvertExpenseCurrency() throws IOException {
        Expense expense = new Expense("Book", "01/04/2023", "Other", 15.0, "USD");
        mockManager.addExpense(expense);
        doNothing().when(mockService).convertExpenseCurrency(expense, "EUR");
        controller.convertExpenseCurrency(expense, "EUR");
        verify(controller.expenseManager.getService(), times(1)).convertExpenseCurrency(expense, "EUR");
    }
}
-----------------------------------------------------------------------------------------------------


##ExpenseFileHandlerTest.java




import com.example.model.Expense;
import com.example.model.ExpenseManager;
import com.example.service.ExpenseService;
import com.example.utils.ExpenseFileHandler;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.File;
import java.io.IOException;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;

// A class for testing with JUnit. I have provided more details in the documentation.
public class ExpenseFileHandlerTest {
    private final String testFileName = "testExpenses.ser"; // Using a test file to avoid overwriting real data
    private ExpenseFileHandler handler;
    private final ExpenseService expenseService = new ExpenseService();
    private final ExpenseFileHandler expenseFileHandler = new ExpenseFileHandler();
    private final ExpenseManager testExpenseManager = new ExpenseManager(expenseService, expenseFileHandler);

    @BeforeEach
    public void setUp() throws IOException {
        handler = new ExpenseFileHandler(testFileName);
        testExpenseManager.addExpense(new Expense("Lunch", "10/10/2023", "Food", 2300, "HUF"));
        testExpenseManager.addExpense(new Expense("Netflix", "11/10/2023", "Entertainment", 5750, "HUF"));
        // Save the test expenses to the file
        handler.saveExpensesToFile(testExpenseManager.getAllExpenses());
    }

    @Test
    public void testSaveExpensesToFile() throws IOException, ClassNotFoundException {
        // Load the expenses from the test file
        List<Expense> loadedExpenses = handler.loadExpensesFromFile();
        // Check that the loaded expenses match what was saved
        assertEquals(testExpenseManager.getAllExpenses().size(), loadedExpenses.size());
        for (int i = 0; i < testExpenseManager.getAllExpenses().size(); i++) {
            assertEquals(testExpenseManager.getAllExpenses().get(i), loadedExpenses.get(i));
        }
    }

    @Test
    public void testLoadExpensesFromFile() throws IOException, ClassNotFoundException {
        // Directly load the expenses from the test file
        List<Expense> loadedExpenses = handler.loadExpensesFromFile();
        // Check that the loaded expenses match the test data
        assertEquals(testExpenseManager.getAllExpenses().size(), loadedExpenses.size());
        for (int i = 0; i < testExpenseManager.getAllExpenses().size(); i++) {
            assertEquals(testExpenseManager.getAllExpenses().get(i).getName(), loadedExpenses.get(i).getName());
            assertEquals(testExpenseManager.getAllExpenses().get(i).getDate(), loadedExpenses.get(i).getDate());
            assertEquals(testExpenseManager.getAllExpenses().get(i).getCategory(), loadedExpenses.get(i).getCategory());
            assertEquals(testExpenseManager.getAllExpenses().get(i).getAmount(), loadedExpenses.get(i).getAmount());
        }
    }

    @AfterEach
    public void tearDown() {
        // Clean up the test file
        new File(testFileName).delete();
    }
}
-----------------------------------------------------------------------------------------------------


##ExpenseManagerTest.java


import com.example.model.Expense;
import com.example.model.ExpenseManager;
import com.example.service.ExpenseService;
import com.example.utils.ExpenseFileHandler;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.ByteArrayOutputStream;
import java.io.PrintStream;
import java.time.YearMonth;
import java.util.List;
import java.util.TreeMap;

import static org.junit.jupiter.api.Assertions.*;

// A class for testing with JUnit. I have provided more details in the documentation.
public class ExpenseManagerTest {

    private ExpenseManager expenseManager;
    private ExpenseService expenseService;
    private ExpenseFileHandler expenseFileHandler;

    // Setting up the expenses before each test. I decided to add three expenses, each of different names, dates, amounts and categories.
    @BeforeEach
    void setUp() {
        expenseService = new ExpenseService();
        expenseFileHandler = new ExpenseFileHandler();
        expenseManager = new ExpenseManager(expenseService, expenseFileHandler);
        // Add some default expenses to the manager
        expenseManager.addExpense(new Expense("Lunch", "26/11/2023", "Food", 6200, "HUF"));
        expenseManager.addExpense(new Expense("Bus Ticket", "24/11/2023", "Transportation", 3500, "HUF"));
        expenseManager.addExpense(new Expense("Cheese", "11/10/2023", "Groceries", 4540, "HUF"));
    }

    @Test
    public void testAddExpense() {
        Expense newExpense = new Expense("Coffee", "20/11/2023", "Food", 600, "HUF");
        expenseManager.addExpense(newExpense);
        assertTrue(expenseManager.getAllExpenses().contains(newExpense));
    }

    @Test
    public void testRemoveExpense() {
        Expense expenseToRemove = new Expense("Lunch", "26/11/2023", "Food", 6200, "HUF");
        assertTrue(expenseManager.removeExpense(expenseManager.getAllExpenses().indexOf(expenseToRemove)));
        assertFalse(expenseManager.getAllExpenses().contains(expenseToRemove));
    }

    @Test
    public void testGetExpensesByCategory() {
        List<Expense> foodExpenses = expenseManager.getExpensesByCategory("Food");
        assertEquals(1, foodExpenses.size()); // Assuming only 1 food expense was added in the setup.
    }

    @Test
    public void testAddInvalidExpense() {
        Exception exception = assertThrows(IllegalArgumentException.class, () -> expenseManager.addExpense(new Expense("", "invalid-date", "Food", -10.0, "HUF")));
        String expectedMessage = "Invalid date format: " + "invalid-date" + ", please use dd/MM/yyyy.";
        String actualMessage = exception.getMessage();
        assertTrue(actualMessage.contains(expectedMessage));
    }

    @Test
    public void testRemoveNonExistentExpense() {
        Expense nonExistentExpense = new Expense("Dinner", "07/04/2005", "Food", 9000, "HUF");
        assertFalse(expenseManager.removeExpense(expenseManager.getAllExpenses().indexOf(nonExistentExpense))); // Should return false as it doesn't exist
    }

    @Test
    public void testEditExpense() {
        Expense originalExpense = new Expense("Lunch", "26/11/2023", "Food", 6200, "HUF");
        Expense updatedExpense = new Expense("Lunch", "26/11/2023", "Food", 4500, "HUF"); // Decreased the amount
        expenseManager.editExpense(originalExpense, updatedExpense);
        assertTrue(expenseManager.getAllExpenses().contains(updatedExpense));
        assertFalse(expenseManager.getAllExpenses().contains(originalExpense));
    }

    @Test
    public void testGetExpensesGroupedByMonth() {
        TreeMap<YearMonth, List<Expense>> groupedExpenses = expenseManager.getExpensesGroupedByMonth();
        assertNotNull(groupedExpenses);
        assertFalse(groupedExpenses.isEmpty());
        // Assuming that you have expenses from different months in the setup, assert that:
        assertEquals(2, groupedExpenses.size()); // Since we have expenses tracked in two different months.
    }

    @Test
    public void testClearExpenses() {
        assertFalse(expenseManager.getAllExpenses().isEmpty());
        expenseManager.clearExpenses();
        assertTrue(expenseManager.getAllExpenses().isEmpty());
    }

    @Test
    public void testEqualsAndHashCode() {
        ExpenseManager anotherManager = new ExpenseManager(expenseService, expenseFileHandler);
        anotherManager.addExpense(new Expense("Lunch", "26/11/2023", "Food", 6200, "HUF"));
        anotherManager.addExpense(new Expense("Bus Ticket", "24/11/2023", "Transportation", 3500, "HUF"));
        anotherManager.addExpense(new Expense("Cheese", "11/10/2023", "Groceries", 4540, "HUF"));

        assertEquals(expenseManager, anotherManager);
        assertEquals(expenseManager.hashCode(), anotherManager.hashCode());

        anotherManager.addExpense(new Expense("Extra", "12/11/2023", "Other", 100, "HUF"));
        assertNotEquals(expenseManager, anotherManager);
    }

    // This test method assumes that you will capture the print stream output.
    @Test
    public void testPrintExpenses() {
        ByteArrayOutputStream outContent = new ByteArrayOutputStream();
        System.setOut(new PrintStream(outContent));

        expenseManager.printExpenses();
        String printedContent = outContent.toString();
        assertTrue(printedContent.contains("Lunch"));
        assertTrue(printedContent.contains("Food"));
        assertTrue(printedContent.contains("6200"));

        // Reset the standard system output to its original.
        System.setOut(System.out);
    }
}
