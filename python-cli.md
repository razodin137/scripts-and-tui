#!/usr/bin/env python3
import sqlite3
import click
import json
import os
from datetime import datetime
from typing import Dict, List
import openai
from pathlib import Path

class ContentManager:
    def __init__(self, db_path: str = "content.db"):
        self.db_path = db_path
        self.setup_database()
        
    def setup_database(self):
        """Initialize SQLite database with required tables"""
        conn = sqlite3.connect(self.db_path)
        c = conn.cursor()
        
        # Create tables for content management
        c.executescript('''
            CREATE TABLE IF NOT EXISTS posts (
                id INTEGER PRIMARY KEY,
                content TEXT NOT NULL,
                platform TEXT NOT NULL,
                status TEXT NOT NULL,
                publish_date DATETIME,
                metrics TEXT,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP
            );
            
            CREATE TABLE IF NOT EXISTS generated_content (
                id INTEGER PRIMARY KEY,
                original_post_id INTEGER,
                content TEXT NOT NULL,
                prompt TEXT NOT NULL,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (original_post_id) REFERENCES posts (id)
            );
            
            CREATE TABLE IF NOT EXISTS analytics (
                id INTEGER PRIMARY KEY,
                post_id INTEGER,
                metric_type TEXT NOT NULL,
                metric_value TEXT NOT NULL,
                recorded_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (post_id) REFERENCES posts (id)
            );
        ''')
        conn.commit()
        conn.close()

    async def generate_content_variations(self, content: str, platform: str) -> List[Dict]:
        """Generate content variations using LLM"""
        prompts = {
            "twitter": [
                "Convert this into a thread: ",
                "Create a poll based on this content: ",
                "Generate a follow-up tweet: "
            ],
            "instagram": [
                "Convert this into carousel slides: ",
                "Generate hashtags for this post: ",
                "Create a story series from this: "
            ],
            "linkedin": [
                "Expand this into a detailed post: ",
                "Create a professional insight based on this: ",
                "Generate discussion points from this: "
            ]
        }
        
        variations = []
        platform_prompts = prompts.get(platform, prompts["twitter"])
        
        for prompt in platform_prompts:
            response = await openai.ChatCompletion.acreate(
                model="gpt-4",
                messages=[
                    {"role": "system", "content": "You are a professional content creator."},
                    {"role": "user", "content": f"{prompt}{content}"}
                ]
            )
            variations.append({
                "prompt": prompt,
                "content": response.choices[0].message.content,
                "platform": platform
            })
            
        return variations

    def save_post(self, content: str, platform: str, status: str = "draft") -> int:
        """Save post to database"""
        conn = sqlite3.connect(self.db_path)
        c = conn.cursor()
        c.execute('''
            INSERT INTO posts (content, platform, status, publish_date)
            VALUES (?, ?, ?, ?)
        ''', (content, platform, status, datetime.now()))
        post_id = c.lastrowid
        conn.commit()
        conn.close()
        return post_id

    def save_generated_content(self, post_id: int, content: str, prompt: str):
        """Save LLM-generated content"""
        conn = sqlite3.connect(self.db_path)
        c = conn.cursor()
        c.execute('''
            INSERT INTO generated_content (original_post_id, content, prompt)
            VALUES (?, ?, ?)
        ''', (post_id, content, prompt))
        conn.commit()
        conn.close()

@click.group()
def cli():
    """Content Automation CLI"""
    pass

@cli.command()
@click.option('--platform', type=click.Choice(['twitter', 'instagram', 'linkedin']), required=True)
@click.option('--content', prompt='Enter your content', help='Content to be published')
@click.option('--schedule', type=click.DateTime(), help='Schedule post for later')
async def create(platform: str, content: str, schedule: datetime = None):
    """Create new content"""
    manager = ContentManager()
    
    # Save original content
    post_id = manager.save_post(content, platform)
    
    # Generate variations
    variations = await manager.generate_content_variations(content, platform)
    
    # Save generated variations
    for variation in variations:
        manager.save_generated_content(
            post_id,
            variation['content'],
            variation['prompt']
        )
    
    click.echo(f"Content created for {platform} with ID: {post_id}")
    click.echo(f"Generated {len(variations)} variations")

@cli.command()
@click.option('--days', default=7, help='Number of days of analytics to export')
def export_analytics(days: int):
    """Export analytics data for PHP dashboard"""
    manager = ContentManager()
    # Implementation for analytics export
    pass

if __name__ == '__main__':
    cli()