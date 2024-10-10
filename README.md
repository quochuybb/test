//pub mod controllers;
//pub mod models;
//pub mod views;
use std::time::{Duration};
use ratatui::{
    backend::CrosstermBackend,
    widgets::{Block, Borders, Paragraph},
    layout::{Layout, Constraint, Direction},
    Terminal,
    buffer::Buffer,
    layout::{Alignment, Rect},
    style::Stylize,
    symbols::border,
    text::{Line, Text, Span},
    widgets::{
        block::{Position, Title}, Widget,
    },
    DefaultTerminal, Frame,
};
use crossterm::event::{self, DisableMouseCapture, EnableMouseCapture, Event, KeyCode,KeyEvent, KeyEventKind};
use tokio::time::sleep;
use std::io;
use ratatui::style::{Color, Style};
use sqlx::Either::Left;

#[derive(Debug, Default)]
pub struct App {
    exit: bool,
    score:u8,
    time_left:i32,
    tick_rate:Duration,
    pub input: String,
    pub output: String,
    pub selected_block: usize,
}
impl App {
    async fn run(&mut self, terminal: &mut DefaultTerminal) -> io::Result<()> {
        self.new();
        while !self.exit {
            terminal.draw(|frame| self.draw(frame))?;
            self.handle_events()?;
            sleep(self.tick_rate).await;
            self.time_left -= 1;
        }

        Ok(())
    }
    pub fn next_block(&mut self) {
        self.selected_block = (self.selected_block + 1) % 4; // Cycle through 4 blocks
    }

    pub fn previous_block(&mut self) {
        if self.selected_block == 0 {
            self.selected_block = 3; // Go to the last block
        } else {
            self.selected_block -= 1;
        }
    }
    fn draw(&self, frame: &mut Frame) {
        frame.render_widget(self, frame.area());
    }

    fn handle_events(&mut self) -> io::Result<()> {
        if event::poll(self.tick_rate)? {
            if let Event::Key(key) = event::read()? {
                if key.code == KeyCode::Char('q') {
                    self.exit = true;
                }
            }
        }
        Ok(())
    }
    fn handle_key_event(&mut self, key_event: KeyEvent) {
        match key_event.code {
            KeyCode::Char('q') => self.exit(),
            _ => {}
        }
    }
    fn exit(&mut self) {
        self.exit = true;
    }
    fn new(&mut self){
        self.selected_block = 0;
        self.input = String::new();
        self.output = String::new();
        self.exit = false;
        self.time_left = 20;
        self.tick_rate = Duration::from_secs(1);
    }
}
impl Widget for &App {
    fn render(self, area: Rect, buf: &mut Buffer) {

        let main_area = Layout::default()
            .direction(Direction::Vertical)
            .constraints([
                Constraint::Percentage(5),
                Constraint::Percentage(70),
                Constraint::Percentage(25),
            ].as_ref())
            .split(area);
        let time_and_score_area = Layout::default()
            .direction(Direction::Horizontal)
            .constraints([
                Constraint::Percentage(5),
                Constraint::Percentage(5),
                Constraint::Percentage(20),
            ].as_ref())
            .split(main_area[0]);
        let query_and_question_area = Layout::default()
            .direction(Direction::Horizontal)
            .constraints([
                Constraint::Percentage(70),
                Constraint::Percentage(30)
            ].as_ref())
            .split(main_area[1]);
        let result_and_features_area = Layout::default()
            .direction(Direction::Horizontal)
            .constraints([
                Constraint::Percentage(70),
                Constraint::Percentage(30)
            ].as_ref())
            .split(main_area[2]);
        let chunks = vec![query_and_question_area[0], query_and_question_area[1], result_and_features_area[0],result_and_features_area[1]];
        // let block_input_query = Block::default()
        //     .title("Input Query")
        //     .borders(Borders::ALL);
        // let block_question = Block::default()
        //     .title("Question")
        //     .borders(Borders::ALL);
        // let block_result = Block::default()
        //     .title("Result")
        //     .borders(Borders::ALL);
        // let block_feature = Block::default()
        //     .title("Feature")
        //     .borders(Borders::ALL);
        let blocks = vec![
            (format!("Query"), "Input your query", Color::Red),
            (format!("Question"), "Take me amount of account", Color::Green),
            (format!("Result"), "3", Color::Blue),
            (format!("Feature"), "Option", Color::Cyan),
        ];
        let block_score = Block::default()
            .title("Score")
            .borders(Borders::ALL);
        let block_time_left = Block::default()
            .title("Time left")
            .borders(Borders::ALL);
        let block_hotkey_guide = Block::default()
            .borders(Borders::ALL);
        // block_input_query.render(query_and_question_area[0], buf);
        // block_question.render(query_and_question_area[1], buf);
        // block_result.render(result_and_features_area[0], buf);
        // block_feature.render(result_and_features_area[1], buf);
        for (i, (chunk,(title, content, color))) in chunks.iter().zip(blocks.iter()).enumerate() {
            let block = Block::default()
                .title(title)
                .borders(Borders::ALL)
                .border_style(
                    if self.selected_block == i {
                        Style::default().fg(Color::Yellow) // Highlight selected block
                    } else {
                        Style::default().fg(Color::White) // Normal block
                    },
                );

            let paragraph = Paragraph::new(content)
                .block(block)
                .wrap(ratatui::widgets::Wrap { trim: true });

            paragraph.render(*chunk, buf);
        }
        let score_text = Text::from(vec![Line::from(vec![
            self.score.to_string().green(),
        ])]);
        Paragraph::new(score_text)
            .centered()
            .block(block_score)
            .render(time_and_score_area[0], buf);
        let time_left_text = Text::from(vec![Line::from(vec![
            self.time_left.to_string().green(),
        ])]);
        Paragraph::new(time_left_text)
            .centered()
            .block(block_time_left)
            .render(time_and_score_area[1], buf);
        let hotkey_guide = Text::from(vec![Line::from(vec![
            "Enter: Choose / \u{2192}, \u{2190}: Move / Esc: Open Menu / ".green().into(),
        ])]);
        Paragraph::new(hotkey_guide)
            .centered()
            .block(block_hotkey_guide)
            .render(time_and_score_area[2], buf);

    }
}
#[tokio::main]
async fn main() -> io::Result<()> {
    let mut terminal = ratatui::init();
    let app_result = App::default().run(&mut terminal).await;
    ratatui::restore();
    app_result
}
