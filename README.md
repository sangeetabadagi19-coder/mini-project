#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define WIDTH 60
#define HEIGHT 20
#define MAX_SHAPES 100

// Define Shape Types
typedef enum {
    SHAPE_LINE,
    SHAPE_RECTANGLE,
    SHAPE_CIRCLE,
    SHAPE_TRIANGLE
} ShapeType;

// Define specific shapes data
typedef struct {
    int x1, y1;
    int x2, y2;
} LineData;

typedef struct {
    int x, y;
    int w, h;
} RectData;

typedef struct {
    int cx, cy;
    int r;
} CircleData;

typedef struct {
    int x1, y1;
    int x2, y2;
    int x3, y3;
} TriData;

// Shape wrapper struct
typedef struct {
    int id;
    ShapeType type;
    union {
        LineData line;
        RectData rect;
        CircleData circle;
        TriData tri;
    } data;
} Shape;

// Global State
Shape shapes[MAX_SHAPES];
int numShapes = 0;
int nextShapeId = 1;
char canvas[HEIGHT][WIDTH];

// Helper to safely read an integer from standard input
int getSafeInt(const char* prompt) {
    char buffer[100];
    int value;
    while (1) {
        printf("%s", prompt);
        if (fgets(buffer, sizeof(buffer), stdin) != NULL) {
            // Remove newline
            buffer[strcspn(buffer, "\n")] = '\0';
            if (strlen(buffer) == 0) {
                continue; // Prompt again on empty line
            }
            char* endptr;
            value = (int)strtol(buffer, &endptr, 10);
            if (*endptr == '\0') {
                return value; // Valid integer
            }
        }
        printf("Invalid input. Please enter a valid integer.\n");
    }
}

// Function to plot a single pixel with boundary checking
void plotPixel(char canvas[HEIGHT][WIDTH], int x, int y) {
    if (x >= 0 && x < WIDTH && y >= 0 && y < HEIGHT) {
        canvas[y][x] = '*';
    }
}

// Bresenham's Line Algorithm
void drawLine(char canvas[HEIGHT][WIDTH], int x1, int y1, int x2, int y2) {
    int dx = abs(x2 - x1);
    int dy = abs(y2 - y1);
    int sx = (x1 < x2) ? 1 : -1;
    int sy = (y1 < y2) ? 1 : -1;
    int err = dx - dy;

    while (1) {
        plotPixel(canvas, x1, y1);
        if (x1 == x2 && y1 == y2) {
            break;
        }
        int e2 = 2 * err;
        if (e2 > -dy) {
            err -= dy;
            x1 += sx;
        }
        if (e2 < dx) {
            err += dx;
            y1 += sy;
        }
    }
}

// Rectangle drawing (outlines)
void drawRectangle(char canvas[HEIGHT][WIDTH], int x, int y, int w, int h) {
    if (w <= 0 || h <= 0) return;
    
    // Draw top and bottom horizontal segments
    for (int col = x; col < x + w; col++) {
        plotPixel(canvas, col, y);
        plotPixel(canvas, col, y + h - 1);
    }
    // Draw left and right vertical segments
    for (int row = y; row < y + h; row++) {
        plotPixel(canvas, x, row);
        plotPixel(canvas, x + w - 1, row);
    }
}

// Midpoint Circle Algorithm
void drawCircle(char canvas[HEIGHT][WIDTH], int cx, int cy, int r) {
    if (r < 0) return;
    int x = 0;
    int y = r;
    int d = 3 - 2 * r;

    while (y >= x) {
        plotPixel(canvas, cx + x, cy + y);
        plotPixel(canvas, cx - x, cy + y);
        plotPixel(canvas, cx + x, cy - y);
        plotPixel(canvas, cx - x, cy - y);
        plotPixel(canvas, cx + y, cy + x);
        plotPixel(canvas, cx - y, cy + x);
        plotPixel(canvas, cx + y, cy - x);
        plotPixel(canvas, cx - y, cy - x);

        if (d < 0) {
            d = d + 4 * x + 6;
        } else {
            d = d + 4 * (x - y) + 10;
            y--;
        }
        x++;
    }
}

// Triangle drawing (outlines)
void drawTriangle(char canvas[HEIGHT][WIDTH], int x1, int y1, int x2, int y2, int x3, int y3) {
    drawLine(canvas, x1, y1, x2, y2);
    drawLine(canvas, x2, y2, x3, y3);
    drawLine(canvas, x3, y3, x1, y1);
}

// Re-renders the canvas using active shapes
void renderCanvas() {
    // Fill canvas with background character '_'
    for (int r = 0; r < HEIGHT; r++) {
        for (int c = 0; c < WIDTH; c++) {
            canvas[r][c] = '_';
        }
    }

    // Render each shape onto the canvas
    for (int i = 0; i < numShapes; i++) {
        Shape s = shapes[i];
        switch (s.type) {
            case SHAPE_LINE:
                drawLine(canvas, s.data.line.x1, s.data.line.y1, s.data.line.x2, s.data.line.y2);
                break;
            case SHAPE_RECTANGLE:
                drawRectangle(canvas, s.data.rect.x, s.data.rect.y, s.data.rect.w, s.data.rect.h);
                break;
            case SHAPE_CIRCLE:
                drawCircle(canvas, s.data.circle.cx, s.data.circle.cy, s.data.circle.r);
                break;
            case SHAPE_TRIANGLE:
                drawTriangle(canvas, s.data.tri.x1, s.data.tri.y1, s.data.tri.x2, s.data.tri.y2, s.data.tri.x3, s.data.tri.y3);
                break;
        }
    }
}

// Displays canvas to terminal inside a clean border box
void displayCanvas() {
    renderCanvas();
    
    // Top border
    printf("+");
    for (int c = 0; c < WIDTH; c++) printf("-");
    printf("+\n");

    // Canvas contents
    for (int r = 0; r < HEIGHT; r++) {
        printf("|");
        for (int c = 0; c < WIDTH; c++) {
            printf("%c", canvas[r][c]);
        }
        printf("|\n");
    }

    // Bottom border
    printf("+");
    for (int c = 0; c < WIDTH; c++) printf("-");
    printf("+\n");
}

// Lists active shapes with details and IDs
void listShapes() {
    if (numShapes == 0) {
        printf("The canvas is currently empty.\n");
        return;
    }
    printf("\n=== Active Shapes ===\n");
    for (int i = 0; i < numShapes; i++) {
        Shape s = shapes[i];
        printf("[%d] ", s.id);
        switch (s.type) {
            case SHAPE_LINE:
                printf("Line: (%d, %d) -> (%d, %d)\n", s.data.line.x1, s.data.line.y1, s.data.line.x2, s.data.line.y2);
                break;
            case SHAPE_RECTANGLE:
                printf("Rectangle: top-left (%d, %d), width %d, height %d\n", s.data.rect.x, s.data.rect.y, s.data.rect.w, s.data.rect.h);
                break;
            case SHAPE_CIRCLE:
                printf("Circle: center (%d, %d), radius %d\n", s.data.circle.cx, s.data.circle.cy, s.data.circle.r);
                break;
            case SHAPE_TRIANGLE:
                printf("Triangle: vertices (%d, %d), (%d, %d), (%d, %d)\n", 
                       s.data.tri.x1, s.data.tri.y1, s.data.tri.x2, s.data.tri.y2, s.data.tri.x3, s.data.tri.y3);
                break;
        }
    }
    printf("=====================\n");
}

// Adds a shape to the canvas
void addShapeMenu() {
    if (numShapes >= MAX_SHAPES) {
        printf("Error: Maximum shape limit reached (%d shapes).\n", MAX_SHAPES);
        return;
    }

    printf("\n--- Add a Shape ---\n");
    printf("1. Line\n");
    printf("2. Rectangle\n");
    printf("3. Circle\n");
    printf("4. Triangle\n");
    printf("5. Back to Main Menu\n");
    int choice = getSafeInt("Enter choice (1-5): ");

    Shape s;
    s.id = nextShapeId;

    switch (choice) {
        case 1:
            s.type = SHAPE_LINE;
            s.data.line.x1 = getSafeInt("Enter start X coordinate (0-59): ");
            s.data.line.y1 = getSafeInt("Enter start Y coordinate (0-19): ");
            s.data.line.x2 = getSafeInt("Enter end X coordinate (0-59): ");
            s.data.line.y2 = getSafeInt("Enter end Y coordinate (0-19): ");
            break;
        case 2:
            s.type = SHAPE_RECTANGLE;
            s.data.rect.x = getSafeInt("Enter top-left X coordinate (0-59): ");
            s.data.rect.y = getSafeInt("Enter top-left Y coordinate (0-19): ");
            s.data.rect.w = getSafeInt("Enter width (1-60): ");
            s.data.rect.h = getSafeInt("Enter height (1-20): ");
            break;
        case 3:
            s.type = SHAPE_CIRCLE;
            s.data.circle.cx = getSafeInt("Enter center X coordinate (0-59): ");
            s.data.circle.cy = getSafeInt("Enter center Y coordinate (0-19): ");
            s.data.circle.r  = getSafeInt("Enter radius: ");
            break;
        case 4:
            s.type = SHAPE_TRIANGLE;
            s.data.tri.x1 = getSafeInt("Enter point 1 X: ");
            s.data.tri.y1 = getSafeInt("Enter point 1 Y: ");
            s.data.tri.x2 = getSafeInt("Enter point 2 X: ");
            s.data.tri.y2 = getSafeInt("Enter point 2 Y: ");
            s.data.tri.x3 = getSafeInt("Enter point 3 X: ");
            s.data.tri.y3 = getSafeInt("Enter point 3 Y: ");
            break;
        case 5:
            return;
        default:
            printf("Invalid choice. Returning to main menu.\n");
            return;
    }

    shapes[numShapes++] = s;
    nextShapeId++;
    printf("Shape added successfully with ID [%d]!\n", s.id);
}

// Deletes a shape from the canvas
void deleteShapeMenu() {
    if (numShapes == 0) {
        printf("No shapes available to delete.\n");
        return;
    }

    listShapes();
    int targetId = getSafeInt("Enter the ID of the shape to delete (or -1 to cancel): ");
    if (targetId == -1) {
        return;
    }

    int foundIndex = -1;
    for (int i = 0; i < numShapes; i++) {
        if (shapes[i].id == targetId) {
            foundIndex = i;
            break;
        }
    }

    if (foundIndex == -1) {
        printf("Shape ID [%d] not found.\n", targetId);
        return;
    }

    // Shift elements left to maintain contiguous array
    for (int i = foundIndex; i < numShapes - 1; i++) {
        shapes[i] = shapes[i + 1];
    }
    numShapes--;
    printf("Shape [%d] deleted successfully!\n", targetId);
}

int main() {
    printf("=== 2D ASCII Graphics Editor ===\n");
    printf("Canvas Resolution: %dx%d (X: 0-%d, Y: 0-%d)\n", WIDTH, HEIGHT, WIDTH-1, HEIGHT-1);

    while (1) {
        printf("\n--- Main Menu ---\n");
        printf("1. Add an object\n");
        printf("2. Delete an object\n");
        printf("3. Display the picture\n");
        printf("4. List current objects\n");
        printf("5. Clear all objects\n");
        printf("6. Exit\n");
        int choice = getSafeInt("Select option (1-6): ");

        switch (choice) {
            case 1:
                addShapeMenu();
                break;
            case 2:
                deleteShapeMenu();
                break;
            case 3:
                displayCanvas();
                break;
            case 4:
                listShapes();
                break;
            case 5:
                numShapes = 0;
                printf("Canvas cleared of all shapes.\n");
                break;
            case 6:
                printf("Exiting editor. Goodbye!\n");
                return 0;
            default:
                printf("Invalid choice. Please select 1 to 6.\n");
                break;
        }
    }
}
