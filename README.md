# project_lavarinth
#include "mbed.h"
#include "stm32f413h_discovery_ts.h"
#include "stm32f413h_discovery_lcd.h"
#include <ctime>
#include <stack>

// predefining some values so that, in case of need, they can be easily changed up

#define max_size 24
#define background_color LCD_COLOR_DARKGREEN
#define text_color LCD_COLOR_WHITE
#define lava_color LCD_COLOR_ORANGE

// declaring needed variables

struct cell
{
  bool bottom_wall, right_wall, visited;
};

bool in_game, random_delay, old_state [max_size] [max_size];
char player_x_position, player_y_position;
unsigned short size, lava_start_delay, lava_spread_delay, width, height,
               current_menu, menu_x_position, menu_y_position;
uint16_t player_color;
cell labyrinth [max_size] [max_size];

InterruptIn left (p5), down (p6), up (p7), right (p8), confirm (p9);

Timeout waiting_line;

// used for setting up the main menu

void open_main_menu ()
{
  menu_x_position = 60;
  menu_y_position = 69 + 25 * ((current_menu - 1) % 4);
  current_menu = 1;
  BSP_LCD_Clear (background_color);
  BSP_LCD_SetBackColor (background_color);
  BSP_LCD_SetTextColor (lava_color);
  BSP_LCD_SetFont (&Font16);
  BSP_LCD_DisplayStringAt (3, 25, (uint8_t*) "LAVArinth", CENTER_MODE);
  BSP_LCD_SetTextColor (text_color);
  BSP_LCD_SetFont (&Font12);
  BSP_LCD_DisplayStringAt (0, 65, (uint8_t*) "New game", CENTER_MODE);
  BSP_LCD_DisplayStringAt (0, 90, (uint8_t*) "Player color", CENTER_MODE);
  BSP_LCD_DisplayStringAt (0, 115, (uint8_t*) "Difficulty", CENTER_MODE);
  BSP_LCD_DisplayStringAt (0, 140, (uint8_t*) "Spread speed", CENTER_MODE);
  BSP_LCD_DisplayStringAt (0, 165, (uint8_t*) "Exit", CENTER_MODE);
  BSP_LCD_DisplayStringAt (5, 225, (uint8_t*) "Enesa Hrustic", LEFT_MODE);
  BSP_LCD_DisplayStringAt (3, 225, (uint8_t*) "Kemal Lipovaca", RIGHT_MODE);
  BSP_LCD_SetTextColor (player_color);
  BSP_LCD_FillCircle (menu_x_position, menu_y_position, 3);
}

// needed for labyrinth drawing (calls each time the difficulty is changed)

void adjust_cell_dimensions ()
{
  width = BSP_LCD_GetXSize () / size;
  height = BSP_LCD_GetYSize () / size;
}

// preparation for labyrinth generation

void initialize_labyrinth ()
{
  for (char i=0; i < size; i++)
    for (char j=0; j < size; j++)
    {
      labyrinth [i] [j].bottom_wall = labyrinth [i] [j].right_wall = true;
      labyrinth [i] [j].visited = false;
    }
}

// generates a size*size labyrinth by visiting a random neighbor cell once starting from a random cell
// at the start, all (bottom and right) walls exist
// by choosing the random neighboring cell, the wall that connects them is removed

void generate_labyrinth ()
{
  initialize_labyrinth ();
  char x_position, y_position;
  x_position = rand () % size;
  y_position = rand () % size;
  labyrinth [x_position] [y_position].visited = true;
  
  // stacks are used to remember which cells have been visited
  // when the last visited cell has no more unvisited neighboring cells, it's coordinates get removed from the stacks
  
  std::stack <char> x_path, y_path;
  x_path.push (x_position);
  y_path.push (y_position);
  while (!x_path.empty () && !y_path.empty ())
    if ((y_position != 0 && labyrinth [x_position] [y_position-1].visited == false && labyrinth [x_position] [y_position-1].right_wall == true) ||
        (x_position != size - 1 && labyrinth [x_position+1] [y_position].visited == false && labyrinth [x_position] [y_position].bottom_wall == true) ||
        (x_position != 0 && labyrinth [x_position-1] [y_position].visited == false && labyrinth [x_position-1] [y_position].bottom_wall == true) ||
        (y_position != size - 1 && labyrinth [x_position] [y_position+1].visited == false && labyrinth [x_position] [y_position].right_wall == true))
    {
      char direction = rand () % 4;
      
      // 0 - left, 1 - down, 2 - up, 3 - right
      // by using only two walls, index manipulation is needed for checking the visited and wall states
      
      if (direction == 0 && y_position != 0 && labyrinth [x_position] [y_position-1].visited == false)
      {
        y_position--;
        labyrinth [x_position] [y_position].right_wall = false;
        labyrinth [x_position] [y_position].visited = true;
        x_path.push (x_position);
        y_path.push (y_position);
      }
      else if (direction == 1 && x_position != size - 1 && labyrinth [x_position+1] [y_position].visited == false)
      {
        labyrinth [x_position] [y_position].bottom_wall = false;
        x_position++;
        labyrinth [x_position] [y_position].visited = true;
        x_path.push (x_position);
        y_path.push (y_position);
      }
      else if (direction == 2 && x_position != 0 && labyrinth [x_position-1] [y_position].visited == false)
      {
        x_position--;
        labyrinth [x_position] [y_position].bottom_wall = false;
        labyrinth [x_position] [y_position].visited = true;
        x_path.push (x_position);
        y_path.push (y_position);
      }
      else if (direction == 3 && y_position != size - 1 && labyrinth [x_position] [y_position+1].visited == false)
      {
        labyrinth [x_position] [y_position].right_wall = false;
        y_position++;
        labyrinth [x_position] [y_position].visited = true;
        x_path.push (x_position);
        y_path.push (y_position);
      }
      else
        continue;
    }
    else
    {
      x_position = x_path.top ();
      y_position = y_path.top ();
      x_path.pop ();
      y_path.pop ();
    }

  // at the end of this process, randomly creates the starting square and ending square
  // the start point can be on the top or on the left side of the labyrinth
  // end square is placed symmetrically from the starting square with respect to the center

  bool start_from_top = rand () % 2;
  if (start_from_top)
  {
    x_position = 0;
    y_position = rand () % size;
    labyrinth [size-1] [size-y_position-1].bottom_wall = false;
  }
  else
  {
    x_position = rand () % size;
    y_position = 0;
    labyrinth [size-x_position-1] [size-1].right_wall = false;
  }
}

// used for getting the position of the starting (and ending) square

void starting_position (char &x_position, char &y_position)
{
  for (char i=0; i < size; i++)
  {
    if (labyrinth [size - 1] [i].bottom_wall == false)
    {
      x_position = 0;
      y_position = size - i - 1;
      break;
    }
    if (labyrinth [i] [size - 1].right_wall == false)
    {
      x_position = size - i - 1;
      y_position = 0;
      break;
    }
  }
}

// simply put, it's a slight variation of the generate_labyrinth algorithm
// two extra stacks are needed because the solution uses the visited state checker,
// but because of the random movement, extra cells will be marked as visited
// one way to handle that is to remember the x-y coordinates of the incorrect cells
// and later change their visited state to false

std::pair <std::stack <char>, std::stack <char> > generate_solution ()
{
  char x_position, y_position, x_ending, y_ending;
  for (char i=0; i < size; i++)
    for (char j=0; j < size; j++)
      labyrinth [i] [j].visited = false;
  starting_position (x_position, y_position);
  x_ending = size - x_position - 1;
  y_ending = size - y_position - 1;
  labyrinth [x_position] [y_position].visited = true;
  std::stack <char> x_path, y_path, x_removal, y_removal;
  x_path.push (x_position);
  y_path.push (y_position);
  while (x_path.top () != x_ending || y_path.top () != y_ending)
  {
    if ((y_position != 0 && labyrinth [x_position] [y_position-1].visited == false && labyrinth [x_position] [y_position-1].right_wall == false) ||
        (x_position != size - 1 && labyrinth [x_position+1] [y_position].visited == false && labyrinth [x_position] [y_position].bottom_wall == false) ||
        (x_position != 0 && labyrinth [x_position-1] [y_position].visited == false && labyrinth [x_position-1] [y_position].bottom_wall == false) ||
        (y_position != size - 1 && labyrinth [x_position] [y_position+1].visited == false && labyrinth [x_position] [y_position].right_wall == false))
    {
      char direction = rand () % 4;
      if (direction == 0 && y_position != 0 && labyrinth [x_position] [y_position-1].visited == false && labyrinth [x_position] [y_position-1].right_wall == false)
      {
        y_position--;
        labyrinth [x_position] [y_position].visited = true;
        x_path.push (x_position);
        y_path.push (y_position);
      }
      else if (direction == 1 && x_position != size - 1 && labyrinth [x_position+1] [y_position].visited == false && labyrinth [x_position] [y_position].bottom_wall == false)
      {
        x_position++;
        labyrinth [x_position] [y_position].visited = true;
        x_path.push (x_position);
        y_path.push (y_position);
      }
      else if (direction == 2 && x_position != 0 && labyrinth [x_position-1] [y_position].visited == false && labyrinth [x_position-1] [y_position].bottom_wall == false)
      {
        x_position--;
        labyrinth [x_position] [y_position].visited = true;
        x_path.push (x_position);
        y_path.push (y_position);
      }
      else if (direction == 3 && y_position != size - 1 && labyrinth [x_position] [y_position+1].visited == false && labyrinth [x_position] [y_position].right_wall == false)
      {
        y_position++;
        labyrinth [x_position] [y_position].visited = true;
        x_path.push (x_position);
        y_path.push (y_position);
      }
      else
        continue;
    }
    else
    {
      x_removal.push (x_position);
      y_removal.push (y_position);
      x_path.pop ();
      y_path.pop ();
      x_position = x_path.top ();
      y_position = y_path.top ();
    }
  }
  while (!x_removal.empty () || !y_removal.empty ())
  {
    labyrinth [x_removal.top ()] [y_removal.top ()].visited = false;
    x_removal.pop ();
    y_removal.pop ();
  }
  return std::make_pair (x_path, y_path);
}

// since top and left walls aren't used, they are first drawn
// the starting and ending point are drawn with the selected player color

void draw_labyrinth ()
{
  BSP_LCD_Clear (background_color);
  BSP_LCD_SetTextColor (text_color);
  BSP_LCD_DrawHLine (0, 0, 240);
  BSP_LCD_DrawVLine (0, 0, 240);
  for (char i=0; i < size; i++)
    for (char j=0; j < size; j++)
    {
      if (labyrinth [i] [j].bottom_wall == true)
        BSP_LCD_DrawHLine (j * width, (i+1) * height - (i == size - 1), width + 1);
      if (labyrinth [i] [j].right_wall == true)
        BSP_LCD_DrawVLine ((j+1) * width - (j == size - 1), i * height, height + 1);
    }
  BSP_LCD_SetTextColor (player_color);
  char x_position, y_position;
  starting_position (x_position, y_position);
  if (x_position == 0 && labyrinth [size-1] [size-y_position-1].bottom_wall == false)
  {
    BSP_LCD_DrawHLine (y_position * width + 1, 0, width - 1 - (y_position == 0));
    BSP_LCD_DrawHLine (BSP_LCD_GetXSize () - (y_position + 1) * width + 1, BSP_LCD_GetYSize () - 1, width - 1 - (y_position == 0));
  }
  else
  {
    BSP_LCD_DrawVLine (0, x_position * height + 1, height - 1 - (x_position == 0));
    BSP_LCD_DrawVLine (BSP_LCD_GetXSize () - 1, BSP_LCD_GetYSize () - (x_position + 1) * height + 1, height - 1 - (x_position == 0));
  }
}

// using the coordinates of the cells that form the path from the beginning to the end acquired
// from the generate_solution function, draws the correct path
// the wait function at the end is used so the player has enough time to analyze the path

void draw_solution ()
{
  char x_position, y_position;
  draw_labyrinth ();
  std::pair <std::stack <char>, std::stack <char> > path = generate_solution ();
  while (!path.first.empty () && !path.second.empty ())
  {
    x_position = path.first.top ();
    y_position = path.second.top ();
    path.first.pop ();
    path.second.pop ();
    labyrinth [x_position] [y_position].visited = false;
    if (x_position == size - 1 && labyrinth [x_position] [y_position].bottom_wall == false)
    {
      BSP_LCD_DrawVLine ((2 * y_position + 1) * width / 2, (2 * (size - 1) + 1) * height / 2, height + 1);
      BSP_LCD_DrawVLine ((2 * (size - y_position - 1) + 1) * width / 2, 0, height / 2 + 1);
    }
    else if (y_position == size - 1 && labyrinth [x_position] [y_position].right_wall == false)
    {
      BSP_LCD_DrawHLine ((2 * (size - 1) + 1) * width / 2, (2 * x_position + 1) * height / 2, width + 1);
      BSP_LCD_DrawHLine (0, (2 * (size - x_position - 1) + 1) * height / 2, width / 2 + 1);
    }
    if (y_position != 0 && labyrinth [x_position] [y_position-1].visited == true && labyrinth [x_position] [y_position-1].right_wall == false)
      BSP_LCD_DrawHLine ((2 * (y_position - 1) + 1) * width / 2, (2 * x_position + 1) * height / 2, width + 1);
    else if (x_position != size - 1 && labyrinth [x_position+1] [y_position].visited == true && labyrinth [x_position] [y_position].bottom_wall == false)
      BSP_LCD_DrawVLine ((2 * y_position + 1) * width / 2, (2 * x_position + 1) * height / 2, height + 1);
    else if (x_position != 0 && labyrinth [x_position-1] [y_position].visited == true && labyrinth [x_position-1] [y_position].bottom_wall == false)
      BSP_LCD_DrawVLine ((2 * y_position + 1) * width / 2, (2 * (x_position - 1) + 1) * height / 2, height + 1);
    else if (y_position != size - 1 && labyrinth [x_position] [y_position+1].visited == true && labyrinth [x_position] [y_position].right_wall == false)
      BSP_LCD_DrawHLine ((2 * y_position + 1) * width / 2, (2 * x_position + 1) * height / 2, width + 1);
  }
  labyrinth [x_position] [y_position].visited = false;
  wait (7);
}

// a random message from the given ones in the function is shown each time the player reaches the goal

void victory ()
{
  in_game = false;
  menu_x_position = 13;
  char victory_message [25];
  switch (rand () % 5)
  {
    case (0):
      strcpy (victory_message, "congratulations !");
      break;
    case (1):
      strcpy (victory_message, "well done !");
      break;
    case (2):
      strcpy (victory_message, "you're the champ !");
      break;
    case (3):
      strcpy (victory_message, "awesome job !");
      break;
    case (4):
      strcpy (victory_message, "impressive !");
  }
  BSP_LCD_Clear (background_color);
  BSP_LCD_SetTextColor (text_color);
  BSP_LCD_DisplayStringAt (0, 95, (uint8_t*) "You won,", CENTER_MODE);
  BSP_LCD_DisplayStringAt (0, 110, (uint8_t*) victory_message, CENTER_MODE);
  waiting_line.attach (&open_main_menu, 5);
}

// similarly to the victory function, in case of losing the player gets a random message at the end
// additionally, the player is given the option to view the solution

void defeat ()
{
  in_game = false;
  current_menu = 5;
  menu_x_position = 95;
  menu_y_position = 144;
  char defeat_message [25];
  switch (rand () % 5)
  {
    case (0):
      strcpy (defeat_message, "better luck next time !");
      break;
    case (1):
      strcpy (defeat_message, "too bad :(");
      break;
    case (2):
      strcpy (defeat_message, "keep trying !");
      break;
    case (3):
      strcpy (defeat_message, "unlucky :/");
      break;
    case (4):
      strcpy (defeat_message, "get good ;)");
  }
  BSP_LCD_Clear (background_color);
  BSP_LCD_SetTextColor (text_color);
  BSP_LCD_DisplayStringAt (0, 60, (uint8_t*) "You lost,", CENTER_MODE);
  BSP_LCD_DisplayStringAt (0, 75, (uint8_t*) defeat_message, CENTER_MODE);
  BSP_LCD_DisplayStringAt (0, 110, (uint8_t*) "Show solution ?", CENTER_MODE);
  BSP_LCD_DisplayStringAt (0, 140, (uint8_t*) "Yes", CENTER_MODE);
  BSP_LCD_DisplayStringAt (0, 160, (uint8_t*) "No", CENTER_MODE);
  BSP_LCD_SetTextColor (player_color);
  BSP_LCD_FillCircle (menu_x_position, menu_y_position, 3);
}

// at game start, draws the player in a form of a dot at the starting location
// old_state is used for the next function

void set_player ()
{
  for (char i=0; i < size; i++)
    for (char j=0; j < size; j++)
      old_state [i] [j] = false;
  starting_position (player_x_position, player_y_position);
  BSP_LCD_SetTextColor (player_color);
  BSP_LCD_FillCircle ((2 * player_y_position + 1) * width / 2, (2 * player_x_position + 1) * height / 2, width / 4);
}

// after a grace period (defined by the difficulty the player chooses), lava starts to spread throughout the entire labyrinth
// onto neighboring blocks every lava_spread_delay seconds, where the delay can be altered from the main menu
// also checks if the lava bumped into the player - if yes, the defeat function gets called

void expand ()
{
  if (in_game == false)
    return;
  BSP_LCD_SetTextColor (lava_color);
  char x_position, y_position;
  starting_position (x_position, y_position);
  bool expansion [max_size] [max_size] = {false};
  expansion [x_position] [y_position] = true;
  if (x_position == player_x_position && y_position == player_y_position)
  {
    defeat ();
    return;
  }
  unsigned short unsafe = 1;
  if (x_position == 0 && labyrinth [size-1] [size-y_position-1].bottom_wall == false)
    BSP_LCD_FillRect (y_position * width + 1, 0, width - 1 + (labyrinth [x_position] [y_position].right_wall == false) - (y_position == size - 1), height - (labyrinth [x_position] [y_position].bottom_wall == true));
  else
    BSP_LCD_FillRect (0, x_position * height + 1, width + (labyrinth [x_position] [y_position].right_wall == false), height - 1 - (labyrinth [x_position] [y_position].bottom_wall == true) - (x_position == size - 1));
  BSP_LCD_SetTextColor (text_color);
  BSP_LCD_DrawHLine (y_position * width, (x_position + 1) * height - (x_position == size - 1), 1);
  BSP_LCD_DrawHLine ((y_position + 1) * width, (x_position + 1) * height, 1);
  BSP_LCD_DrawHLine ((y_position + 1) * width - (y_position == size - 1), x_position * height, 1);
  while (unsafe != size * size)
  {
    for (char i=0; i < size; i++)
      for (char j=0; j < size; j++)
        old_state [i] [j] = expansion [i] [j];
    if (random_delay)
      wait_ms (rand () % 1901 + 100);
    else
      wait_ms (lava_spread_delay);
    if (in_game == 0)
      break;
    BSP_LCD_SetTextColor (lava_color);
    for (char i=0; i < size; i++)
      for (char j=0; j < size; j++)
        if (old_state [i] [j] == true)
        {
          if (j != 0 && old_state [i] [j-1] == false && labyrinth [i] [j-1].right_wall == false)
          {
            BSP_LCD_FillRect ((j-1) * width + 1, i * height + 1, width, height - 1 - (labyrinth [i] [j-1].bottom_wall == true) - (i == size - 1));
            expansion [i] [j-1] = true;
            unsafe++;
            if (labyrinth [i] [j-1].bottom_wall == false)
            {
              BSP_LCD_SetTextColor (text_color);
              BSP_LCD_DrawHLine (j * width, (i+1) * height - (i == size - 1), 1);
              BSP_LCD_SetTextColor (lava_color);
            }
            if (i == player_x_position && j == player_y_position + 1)
            {
              defeat ();
              return;
            }
          }
          if (i != size - 1 && old_state [i+1] [j] == false && labyrinth [i] [j].bottom_wall == false)
          {
            BSP_LCD_FillRect (j * width + 1, (i+1) * height + 1, width - (labyrinth [i+1] [j].right_wall == true) - (j == size - 1), height - 1 - (labyrinth [i+1] [j].bottom_wall == true) - (i == size - 2));
            expansion [i+1] [j] = true;
            unsafe++;
            if (labyrinth [i+1] [j].right_wall == false)
            {
              BSP_LCD_SetTextColor (text_color);
              BSP_LCD_DrawVLine ((j+1) * width - (j == size - 1), (i+2) * height - (i == size - 2), 1);
              BSP_LCD_SetTextColor (lava_color);
            }
            if (i == player_x_position - 1 && j == player_y_position)
            {
              defeat ();
              return;
            }
          }
          if (i != 0 && old_state [i-1] [j] == false && labyrinth [i-1] [j].bottom_wall == false)
          {
            BSP_LCD_FillRect (j * width + 1, (i-1) * height + 1, width - (labyrinth [i-1] [j].right_wall == true) - (j == size - 1), height - 1);
            expansion [i-1] [j] = true;
            unsafe++;
            if (labyrinth [i-1] [j].right_wall == false)
            {
              BSP_LCD_SetTextColor (text_color);
              BSP_LCD_DrawVLine ((j+1) * width - (j == size - 1), i * height, 1);
              BSP_LCD_SetTextColor (lava_color);
            }
            if (i == player_x_position + 1 && j == player_y_position)
            {
              defeat ();
              return;
            }
          }
          if (j != size - 1 && old_state [i] [j+1] == false && labyrinth [i] [j].right_wall == false)
          {
            BSP_LCD_FillRect ((j+1) * width + 1, i * height + 1, width - (labyrinth [i] [j+1].right_wall == true) - (j == size - 2), height - 1 - (labyrinth [i] [j+1].bottom_wall == true) - (i == size - 1));
            expansion [i] [j+1] = true;
            unsafe++;
            if (labyrinth [i] [j+1].bottom_wall == false)
            {
              BSP_LCD_SetTextColor (text_color);
              BSP_LCD_DrawHLine ((j+2) * width - (j == size - 2), (i+1) * height - (i == size - 1), 1);
              BSP_LCD_SetTextColor (lava_color);
            }
            if (i == player_x_position && j == player_y_position - 1)
            {
              defeat ();
              return;
            }
          }
        }
  }
}

// the next four functions update the player position in-game
// all of these functions check if the player bumped into the lava
// since the end can be reached by moving down or right, those functions check if the player finished with the maze

void move_left ()
{
  if (in_game == 1 && player_y_position != 0 && labyrinth [player_x_position] [player_y_position-1].right_wall == false)
    if (old_state [player_x_position] [player_y_position-1] == true)
      defeat ();
    else
    {
      BSP_LCD_SetTextColor (background_color);
      BSP_LCD_FillCircle ((2 * player_y_position-- + 1) * width / 2, (2 * player_x_position + 1) * height / 2, width / 4);
      BSP_LCD_SetTextColor (player_color);
      BSP_LCD_FillCircle ((2 * player_y_position + 1) * width / 2, (2 * player_x_position + 1) * height / 2, width / 4);
    }
}

// moving down and up is available in the main menu also, so that's included for them as well

void move_down ()
{
  
  // without this check the user can in some menus move the indicator
  // (or start a new game) which in some cases leads to crashes
  
  if (in_game == 0)
  {
    if (menu_x_position == 13)
      return;
    BSP_LCD_SetTextColor (background_color);
    BSP_LCD_FillCircle (menu_x_position, menu_y_position, 3);
    
    // depending from the current menu, the indicator moves with the player's button presses
    
    switch (current_menu)
    {
      case (1):
        if (menu_y_position < 160)
          menu_y_position += 25;
        else
          menu_y_position = 69;
        break;
      case (2):
        if (menu_y_position < 160)
          menu_y_position += 20;
        else
          menu_y_position = 84;
        break;
      case (3):
        if (menu_y_position < 140)
          menu_y_position += 20;
        else
          menu_y_position = 94;
        break;
      case (4):
        if (menu_y_position < 140)
          menu_y_position += 27;
        else
          menu_y_position = 96;
        break;
      case (5):
        if (menu_y_position < 160)
          menu_y_position += 20;
        else
          menu_y_position = 144;
    }
    BSP_LCD_SetTextColor (player_color);
    BSP_LCD_FillCircle (menu_x_position, menu_y_position, 3);
  }
  else if (labyrinth [player_x_position] [player_y_position].bottom_wall == false)
    if (player_x_position == size - 1)
      victory ();
    else if (old_state [player_x_position+1] [player_y_position] == true)
      defeat ();
    else
    {
      BSP_LCD_SetTextColor (background_color);
      BSP_LCD_FillCircle ((2 *  player_y_position + 1) * width / 2, (2 * player_x_position++ + 1) * height / 2, height / 4);
      BSP_LCD_SetTextColor (player_color);
      BSP_LCD_FillCircle ((2 * player_y_position + 1) * width / 2, (2 * player_x_position + 1) * height / 2, width / 4);
    }
}

void move_up ()
{
  if (in_game == 0)
  {
    if (menu_x_position == 13)
      return;
    BSP_LCD_SetTextColor (background_color);
    BSP_LCD_FillCircle (menu_x_position, menu_y_position, 3);
    switch (current_menu)
    {
      case (1):
        if (menu_y_position > 80)
          menu_y_position -= 25;
        else
          menu_y_position = 169;
        break;
      case (2):
        if (menu_y_position > 100)
          menu_y_position -= 20;
        else
          menu_y_position = 164;
        break;
      case (3):
        if (menu_y_position > 100)
          menu_y_position -= 20;
        else
          menu_y_position = 154;
        break;
      case (4):
        if (menu_y_position > 100)
          menu_y_position -= 27;
        else
          menu_y_position = 150;
        break;
      case (5):
        if (menu_y_position > 160)
          menu_y_position -= 20;
        else
          menu_y_position = 164;
    }
    BSP_LCD_SetTextColor (player_color);
    BSP_LCD_FillCircle (menu_x_position, menu_y_position, 3);
  }
  else if (player_x_position != 0 && labyrinth [player_x_position-1] [player_y_position].bottom_wall == false)
    if (old_state [player_x_position-1] [player_y_position] == true)
      defeat ();
    else 
    {
      BSP_LCD_SetTextColor (background_color);
      BSP_LCD_FillCircle ((2 * player_y_position + 1) * width / 2, (2 * player_x_position-- + 1) * height / 2, height / 4);
      BSP_LCD_SetTextColor (player_color);
      BSP_LCD_FillCircle ((2 * player_y_position + 1) * width / 2, (2 * player_x_position + 1) * height / 2, width / 4);
    }
}

void move_right ()
{
  if (in_game == 1 && labyrinth [player_x_position] [player_y_position].right_wall == false)
    if (player_y_position == size - 1)
      victory ();
    else if (old_state [player_x_position] [player_y_position+1] == true)
      defeat ();
    else
    {
      BSP_LCD_SetTextColor (background_color);
      BSP_LCD_FillCircle ((2 * player_y_position++ + 1) * width / 2, (2 * player_x_position + 1) * height / 2, width / 4);
      BSP_LCD_SetTextColor (player_color);
      BSP_LCD_FillCircle ((2 * player_y_position + 1) * width / 2, (2 * player_x_position + 1) * height / 2, width / 4);
    }
}

// depending from the current state of the program (is the in-game marker up, which menu's
// being accessed currently etc.) checks everything and acts accordingly

void activate ()
{
  switch (current_menu)
  {
    
    // main menu
    
    case (1):
      
      // the player can press the confirm button to give up from solving the labyrinth
      // and, similarly to losing, displays a message and asks to show the solution
      
      if (in_game == 1)
      {
        in_game = false;
        current_menu = 5;
        menu_x_position = 95;
        menu_y_position = 144;
        char gave_up_message [25];
        switch (rand () % 3)
        {
          case (0):
            strcpy (gave_up_message, "what a disappointment.");
            break;
          case (1):
            strcpy (gave_up_message, "coward.");
            break;
          case (2):
            strcpy (gave_up_message, "better RNG next time !");
            break;
          case (3):
            strcpy (gave_up_message, "why tho ?");
            break;
          case (4):
            strcpy (gave_up_message, "but I'm still proud :)");
        }
        BSP_LCD_Clear (background_color);
        BSP_LCD_SetTextColor (text_color);
        BSP_LCD_DisplayStringAt (0, 60, (uint8_t*) "You gave up,", CENTER_MODE);
        BSP_LCD_DisplayStringAt (0, 75, (uint8_t*) gave_up_message, CENTER_MODE);
        BSP_LCD_DisplayStringAt (0, 110, (uint8_t*) "Show solution ?", CENTER_MODE);
        BSP_LCD_DisplayStringAt (0, 140, (uint8_t*) "Yes", CENTER_MODE);
        BSP_LCD_DisplayStringAt (0, 160, (uint8_t*) "No", CENTER_MODE);
        BSP_LCD_SetTextColor (player_color);
        BSP_LCD_FillCircle (menu_x_position, menu_y_position, 3);
      }
      else if (menu_x_position == 60)
        switch (menu_y_position)
        {
          case (69):
            
            // pre-game screen that tells how much time does the player have before
            // the game starts and before the lava starts to spread
            
            menu_x_position = 13;
            generate_labyrinth ();
            BSP_LCD_Clear (background_color);
            BSP_LCD_SetTextColor (text_color);
            for (char i=0; i < 5; i++)
            {
              char lava_start_information [29], game_start_information [30];
              BSP_LCD_DisplayStringAt (0, 75, (uint8_t*) "Lava will start to spread", CENTER_MODE);
              sprintf (lava_start_information, "%hu seconds after game start.", lava_start_delay);
              BSP_LCD_DisplayStringAt (0, 90, (uint8_t*) lava_start_information, CENTER_MODE);
              sprintf (game_start_information, "Game will start in %hu seconds.", 5-i);
              BSP_LCD_DisplayStringAt (0, 120, (uint8_t*) game_start_information, CENTER_MODE);
              BSP_LCD_DisplayStringAt (0, 145, (uint8_t*) "Have fun, good luck !", CENTER_MODE);
              wait (1);
            }
            in_game = true;
            draw_labyrinth ();
            set_player ();
            waiting_line.attach (&expand, lava_start_delay);
            break;
          case (94):
            
            // player color changing menu
            
            current_menu = 2;
            menu_x_position = 80;
            switch (player_color)
            {
              case (LCD_COLOR_RED):
                menu_y_position = 84;
                break;
              case (LCD_COLOR_YELLOW):
                menu_y_position = 104;
                break;
              case (LCD_COLOR_GREEN):
                menu_y_position = 124;
                break;
              case (LCD_COLOR_CYAN):
                menu_y_position = 144;
                break;
              case (LCD_COLOR_MAGENTA):
                menu_y_position = 164;
            }
            BSP_LCD_Clear (background_color);
            BSP_LCD_SetTextColor (text_color);
            BSP_LCD_DisplayStringAt (0, 50, (uint8_t*) "Choose color:", CENTER_MODE);
            BSP_LCD_DisplayStringAt (5, 225, (uint8_t*) "Enesa Hrustic", LEFT_MODE);
            BSP_LCD_DisplayStringAt (3, 225, (uint8_t*) "Kemal Lipovaca", RIGHT_MODE);
            BSP_LCD_SetTextColor (LCD_COLOR_RED);
            BSP_LCD_DisplayStringAt (0, 80, (uint8_t*) "Red", CENTER_MODE);
            BSP_LCD_SetTextColor (LCD_COLOR_YELLOW);
            BSP_LCD_DisplayStringAt (0, 100, (uint8_t*) "Yellow", CENTER_MODE);
            BSP_LCD_SetTextColor (LCD_COLOR_GREEN);
            BSP_LCD_DisplayStringAt (0, 120, (uint8_t*) "Green", CENTER_MODE);
            BSP_LCD_SetTextColor (LCD_COLOR_CYAN);
            BSP_LCD_DisplayStringAt (0, 140, (uint8_t*) "Blue", CENTER_MODE);
            BSP_LCD_SetTextColor (LCD_COLOR_MAGENTA);
            BSP_LCD_DisplayStringAt (0, 160, (uint8_t*) "Pink", CENTER_MODE);
            BSP_LCD_SetTextColor (player_color);
            BSP_LCD_FillCircle (menu_x_position, menu_y_position, 3);
            break;
          case (119):
            
            // difficulty menu
            
            current_menu = 3;
            menu_x_position = 50;
            switch (size)
            {
              case (12):
                menu_y_position = 94;
                break;
              case (16):
                menu_y_position = 114;
                break;
              case (20):
                menu_y_position = 134;
                break;
              case (24):
                menu_y_position = 154;
            }
            BSP_LCD_Clear (background_color);
            BSP_LCD_SetTextColor (text_color);
            BSP_LCD_DisplayStringAt (0, 60, (uint8_t*) "Choose difficulty:", CENTER_MODE);
            BSP_LCD_DisplayStringAt (0, 90, (uint8_t*) "Easy (12x12)", CENTER_MODE);
            BSP_LCD_DisplayStringAt (0, 110, (uint8_t*) "Normal (16x16)", CENTER_MODE);
            BSP_LCD_DisplayStringAt (0, 130, (uint8_t*) "Hard (20x20)", CENTER_MODE);
            BSP_LCD_DisplayStringAt (0, 150, (uint8_t*) "Expert (24x24)", CENTER_MODE);
            BSP_LCD_DisplayStringAt (5, 225, (uint8_t*) "Enesa Hrustic", LEFT_MODE);
            BSP_LCD_DisplayStringAt (3, 225, (uint8_t*) "Kemal Lipovaca", RIGHT_MODE);
            BSP_LCD_SetTextColor (player_color);
            BSP_LCD_FillCircle (menu_x_position, menu_y_position, 3);
            break;
          case (144):
            
            // lava spread speed menu
            
            current_menu = 4;
            menu_x_position = 20;
            switch (lava_spread_delay)
            {
              case (1500):
                menu_y_position = 96;
                break;
              case (500):
                menu_y_position = 150;
            }
            if (random_delay == true)
              menu_y_position = 123;
            BSP_LCD_Clear (background_color);
            BSP_LCD_SetTextColor (text_color);
            BSP_LCD_DisplayStringAt (0, 62, (uint8_t*) "Choose spread speed:", CENTER_MODE);
            BSP_LCD_DisplayStringAt (0, 92, (uint8_t*) "Slow (every 1.5 seconds)", CENTER_MODE);
            BSP_LCD_DisplayStringAt (0, 112, (uint8_t*) "Random (from 0.1 seconds", CENTER_MODE);
            BSP_LCD_DisplayStringAt (0, 126, (uint8_t*) "up to every 2 seconds)", CENTER_MODE);
            BSP_LCD_DisplayStringAt (0, 146, (uint8_t*) "Fast (every 0.5 seconds)", CENTER_MODE);
            BSP_LCD_DisplayStringAt (5, 225, (uint8_t*) "Enesa Hrustic", LEFT_MODE);
            BSP_LCD_DisplayStringAt (3, 225, (uint8_t*) "Kemal Lipovaca", RIGHT_MODE);
            BSP_LCD_SetTextColor (player_color);
            BSP_LCD_FillCircle (menu_x_position, menu_y_position, 3);
            break;
          case (169):
            
            // if the user decides to exit from the game, a goodbye message is displayed
            
            BSP_LCD_Clear (background_color);
            BSP_LCD_SetTextColor (text_color);
            BSP_LCD_DisplayStringAt (5, 100, (uint8_t*) "See you next time !", CENTER_MODE);
            menu_x_position = 13;
            BSP_LCD_DeInit ();
        }
      break;
    
    // the next parts of the code are self-explanatory
    
    case (2):
      switch (menu_y_position)
      {
        case (84):
          player_color = LCD_COLOR_RED;
          break;
        case (104):
          player_color = LCD_COLOR_YELLOW;
          break;
        case (124):
          player_color = LCD_COLOR_GREEN;
          break;
        case (144):
          player_color = LCD_COLOR_CYAN;
          break;
        case (164):
          player_color = LCD_COLOR_MAGENTA;
      }
      open_main_menu ();
      break;
    case (3):
      switch (menu_y_position)
      {
        case (94):
          size = 12;
          lava_start_delay = 5;
          break;
        case (114):
          size = 16;
          lava_start_delay = 7;
          break;
        case (134):
          size = 20;
          lava_start_delay = 10;
          break;
        case (154):
          size = 24;
          lava_start_delay = 15;
      }
      adjust_cell_dimensions ();
      open_main_menu ();
      break;
    case (4):
      switch (menu_y_position)
      {
        case (96):
          lava_spread_delay = 1500;
          random_delay = false;
          break;
        case (123):
          random_delay = true;
          break;
        case (150):
          lava_spread_delay = 500;
          random_delay = false;
      }
      open_main_menu ();
      break;
    case (5):
      
      // menu shown after losing / giving up
      
      if (menu_x_position == 95)
      {
        if (menu_y_position == 144)
        {
          menu_x_position = 13;
          draw_solution ();
        }
        open_main_menu ();
      }
  }
}

// initialization of all needed variables (at the start)
// the ones that are not initialized here are (probably) initialized elsewhere (when needed)

void initialize_program ()
{
  srand (time (NULL));
  in_game = random_delay = false;
  size = 12;
  lava_start_delay = 5;
  lava_spread_delay = 1500;
  width = BSP_LCD_GetXSize () / size;
  height = BSP_LCD_GetYSize () / size;
  current_menu = 1;
  player_color = LCD_COLOR_RED;
  BSP_LCD_Init ();
  BSP_LCD_Clear (background_color);
  BSP_LCD_SetBackColor (background_color);
  BSP_LCD_SetTextColor (text_color);
  BSP_LCD_SetFont (&Font12);
  BSP_LCD_DisplayStringAt (0, 90, (uint8_t*) "Welcome", CENTER_MODE);
  wait (1);
  BSP_LCD_DisplayStringAt (0, 110, (uint8_t*) "to", CENTER_MODE);
  wait (1);
  BSP_LCD_DisplayStringAt (0, 130, (uint8_t*) "the", CENTER_MODE);
  wait (1);
  BSP_LCD_Clear (background_color);
  BSP_LCD_SetTextColor (lava_color);
  BSP_LCD_SetFont (&Font16);
  BSP_LCD_DisplayStringAt (3, 25, (uint8_t*) "LAVArinth", CENTER_MODE);
  wait (2);
  BSP_LCD_SetTextColor (text_color);
  BSP_LCD_SetFont (&Font12);
  BSP_LCD_DisplayStringAt (0, 65, (uint8_t*) "New game", CENTER_MODE);
  wait_ms (300);
  BSP_LCD_DisplayStringAt (0, 90, (uint8_t*) "Player color", CENTER_MODE);
  wait_ms (300);
  BSP_LCD_DisplayStringAt (0, 115, (uint8_t*) "Difficulty", CENTER_MODE);
  wait_ms (300);
  BSP_LCD_DisplayStringAt (0, 140, (uint8_t*) "Spread speed", CENTER_MODE);
  wait_ms (300);
  BSP_LCD_DisplayStringAt (0, 165, (uint8_t*) "Exit", CENTER_MODE);
  wait_ms (300);
  BSP_LCD_DisplayStringAt (5, 225, (uint8_t*) "Enesa Hrustic", LEFT_MODE);
  wait_ms (300);
  BSP_LCD_DisplayStringAt (3, 225, (uint8_t*) "Kemal Lipovaca", RIGHT_MODE);
  wait_ms (300);
  left.rise (&move_left);
  down.rise (&move_down);
  up.rise (&move_up);
  right.rise (&move_right);
  confirm.rise (&activate);
  open_main_menu ();
}

int main ()
{
  initialize_program ();
}
